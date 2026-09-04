<script lang="ts">
    import isEqual from "lodash/isEqual"
    import { tick } from "svelte"
    import { DBState } from 'src/ts/stores.svelte'
    import { sleep } from "src/ts/util"
    import { alertError } from "../../ts/alert"
    import { addMetadataToElement, ParseMarkdown, postTranslationParse, trimMarkdown, type CbsConditions, type simpleCharacterArgument } from "../../ts/parser/parser.svelte"
    import { getLLMCache, translateHTML } from "../../ts/translator/translator"
    import { getModuleAssets } from "src/ts/process/modules";
    import { getCurrentCharacter } from "src/ts/storage/database.svelte";
    import { getFileSrc } from "src/ts/globalApi.svelte";

    interface Props {
        character?: simpleCharacterArgument|string|null
        firstMessage?: boolean
        idx?: number
        msgDisplay?: string
        name?: string
        role: string|null
        translated: boolean
        translating: boolean
        retranslate: boolean
        renderRevision?: number
        bodyRoot?: HTMLElement|null
        modelShortName: string
        renderRawStreaming?: boolean
        rawStreamingText?: string
    }

    let {
        character = null,
        idx = 0,
        firstMessage = false,
        msgDisplay = '',
        role,
        translated = $bindable(false),
        translating = $bindable(false),
        retranslate = $bindable(false),
        renderRevision = 0,
        bodyRoot,
        modelShortName = '',
        renderRawStreaming = false,
        rawStreamingText = '',
    }: Props =  $props()

    // svelte-ignore non_reactive_update
    let lastParsed = ''
    let lastCharArg:string|simpleCharacterArgument = null
    let lastChatId = -10
    let lastRenderedRevision: number | null = null

    function getCbsCondition(){
        try{
            const cbsConditions:CbsConditions = {
                firstmsg: firstMessage ?? false,
                chatRole: role,
            }
            return cbsConditions
        }
        catch(e){
            return {
                firstmsg: firstMessage ?? false,
                chatRole: null,
            }
        }
    }

    let shouldRenderRawStreaming = $derived(renderRawStreaming && !translated && !retranslate)

    // The caller only needs to know whether a candidate is better than maxDistance.
    // Keep two DP rows and only evaluate the band that can still beat that limit.
    const getDistanceBelow = (left: string, right: string, maxDistance: number) => {
        if(left === right){
            return 0
        }
        if(maxDistance <= 0 || Math.abs(left.length - right.length) >= maxDistance){
            return maxDistance
        }
        if(left.length > right.length){
            [left, right] = [right, left]
        }

        let previous = new Uint16Array(right.length + 1)
        let current = new Uint16Array(right.length + 1)
        for(let j = 0; j <= right.length; j++){
            previous[j] = Math.min(j, maxDistance)
        }

        for(let i = 1; i <= left.length; i++){
            current.fill(maxDistance)
            if(i < maxDistance){
                current[0] = i
            }

            const start = Math.max(1, i - maxDistance + 1)
            const end = Math.min(right.length, i + maxDistance - 1)
            let rowMin = maxDistance
            for(let j = start; j <= end; j++){
                const distance = Math.min(
                    previous[j] + 1,
                    current[j - 1] + 1,
                    previous[j - 1] + (left.charCodeAt(i - 1) === right.charCodeAt(j - 1) ? 0 : 1)
                )
                current[j] = distance
                rowMin = Math.min(rowMin, distance)
            }

            if(rowMin >= maxDistance){
                return maxDistance
            }
            ;[previous, current] = [current, previous]
        }

        return Math.min(previous[right.length], maxDistance)
    }

    const markParsing = async (data: string, charArg: string | simpleCharacterArgument, chatID: number, requestedRevision: number, tries?:number) => {
        // track 'translated' and 'retranslate' state
        translated;
        retranslate;
        const preservePendingContent = lastRenderedRevision !== null && requestedRevision !== lastRenderedRevision
        let lastParsedQueue = ''
        let mode = 'notrim' as const
        try {
            if((!isEqual(lastCharArg, charArg)) || (chatID !== lastChatId)){
                lastParsedQueue = ''
                lastCharArg = charArg
                lastChatId = chatID
                let translateText = false
                try {
                    if(DBState.db.autoTranslate){
                        if(DBState.db.autoTranslateCachedOnly && DBState.db.translatorType === 'llm'){
                            const cache = DBState.db.translateBeforeHTMLFormatting
                            ? await getLLMCache(data)
                            : !DBState.db.legacyTranslation
                            ? await getLLMCache(await ParseMarkdown(data, charArg, 'pretranslate', chatID, getCbsCondition()))
                            : await getLLMCache(await ParseMarkdown(data, charArg, mode, chatID, getCbsCondition()))
                  
                            translateText = cache !== null
                        }
                        else{
                            translateText = true
                        }
                    }

                    const lastTranslated = translated

                    setTimeout(() => {
                            translated = translateText
                    }, 10)

                    // State change of `translated` triggers markParsing again,
                    // causing redundant translation attempts
                    if (lastTranslated !== translateText) {
                        return;
                    }
                } catch (error) {
                    console.error(error)
                }
            }
            if(retranslate || translated){
                if (DBState.db.showTranslationLoading && !preservePendingContent) {
                    lastParsed = `<div style="display:flex;justify-content:center;align-items:center;height:48px;"><div style="animation: spin 1s linear infinite; border-radius: 50%; height: 32px; width: 32px; border: 2px solid #3b82f6; border-top: 2px solid transparent;"></div></div><style>@keyframes spin { to { transform: rotate(360deg); } }</style>`
                }

                let transResult
                
                if(DBState.db.translatorType === 'llm' && DBState.db.translateBeforeHTMLFormatting){
                    await sleep(100)
                    translating = true
                    data = await translateHTML(data, false, charArg, chatID, retranslate)
                    translating = false
                    const marked = await ParseMarkdown(data, charArg, mode, chatID, getCbsCondition())
                    lastParsedQueue = marked
                    lastCharArg = charArg
                    transResult = marked
                }
                else if(!DBState.db.legacyTranslation){
                    const marked = await ParseMarkdown(data, charArg, 'pretranslate', chatID, getCbsCondition())
                    translating = true
                    const translated = await postTranslationParse(await translateHTML(marked, false, charArg, chatID, retranslate))
                    translating = false
                    lastParsedQueue = translated
                    lastCharArg = charArg
                    transResult = translated
                }
                else{
                    const marked = await ParseMarkdown(data, charArg, mode, chatID, getCbsCondition())
                    translating = true
                    const translated = await translateHTML(marked, false, charArg, chatID, retranslate)
                    translating = false
                    lastParsedQueue = translated
                    lastCharArg = charArg
                    transResult = translated
                }

                setTimeout(() => {
                    retranslate = false
                }, 10);

                return transResult
            }
            else{
                const marked = await ParseMarkdown(data, charArg, mode, chatID, getCbsCondition())
                lastParsedQueue = marked
                lastCharArg = charArg
                return marked
            }   
        } catch (error) {
            //retry
            if(tries > 2){

                alertError(`Error while parsing chat message: ${translated}, ${error.message}, ${error.stack}`)
                return data
            }
            const retried = await markParsing(data, charArg, chatID, requestedRevision, (tries ?? 0) + 1)
            if (retried !== undefined) {
                lastParsedQueue = retried
            }
            return retried
        }
        finally{
            //since trimMarkdown is fast, we don't need to cache it
            lastParsed = lastParsedQueue
            lastRenderedRevision = requestedRevision
        }
    }

    const checkImg = () => {
        if(!DBState.db.newImageHandlingBeta || !bodyRoot){
            return
        }
        const imgs = bodyRoot.querySelectorAll('img:not([src^="data:"]):not([src^="http:"]):not([src^="https:"]):not([src^="blob:"]):not([src^="file:"]):not([src^="tauri:"]):not([noimage])') as NodeListOf<HTMLImageElement>
        
        if (imgs.length > 0) {
            const currentCharacter = getCurrentCharacter()
            const styl = currentCharacter.prebuiltAssetStyle
            const assets = getModuleAssets().concat(currentCharacter.additionalAssets ?? [])
            const normalizedAssets = assets.map((asset) => {
                return {
                    name: asset[0].toLocaleLowerCase(),
                    path: asset[1]
                }
            })
            const exactAssets = new Map(normalizedAssets.map((asset) => [asset.name, asset.path]))
            const resolvedAssetPaths = new Map<string, string | null>()
            const fileSrcPromises = new Map<string, Promise<string>>()

            const getFileSrcOnce = (path: string) => {
                let pending = fileSrcPromises.get(path)
                if(!pending){
                    pending = getFileSrc(path)
                    fileSrcPromises.set(path, pending)
                }
                return pending
            }

            const findAssetPath = (name: string) => {
                if(resolvedAssetPaths.has(name)){
                    return resolvedAssetPaths.get(name) ?? null
                }

                const exact = exactAssets.get(name)
                if(exact){
                    resolvedAssetPaths.set(name, exact)
                    return exact
                }

                if(name.length < 3){
                    resolvedAssetPaths.set(name, null)
                    return null
                }

                const prefixLoc = name.lastIndexOf('.')
                const prefix = prefixLoc > 0 ? name.substring(0, prefixLoc) : ''
                let currentDistance = 1000
                let currentFound = ''
                for(const asset of normalizedAssets){
                    if(!asset.name.startsWith(prefix)){
                        continue
                    }
                    if(Math.abs(name.length - asset.name.length) >= currentDistance){
                        continue
                    }
                    const distance = getDistanceBelow(name, asset.name, currentDistance)
                    if(distance < currentDistance){
                        currentDistance = distance
                        currentFound = asset.path
                    }
                }

                const resolved = currentFound || null
                resolvedAssetPaths.set(name, resolved)
                return resolved
            }

            imgs.forEach(async (img) => {
                const name = img.getAttribute('src')?.toLocaleLowerCase() || ''

                if(
                    name.length > 200 ||
                    name.includes(':')
                ){
                    img.setAttribute('noimage', 'true')
                    return
                }

                const foundAsset = findAssetPath(name)
                if(!foundAsset){
                    img.setAttribute('noimage', 'true')
                    return
                }

                const got = await getFileSrcOnce(foundAsset)
                const currentName = img.getAttribute('src')?.toLocaleLowerCase() || ''
                if(name !== currentName){
                    return
                }

                img.setAttribute('src', got)
                img.classList.add('root-loaded-image')
                img.classList.add('root-loaded-image-' + styl)
                img.removeAttribute('noimage')
            })
        }
    }

    let markParsingResult = $derived.by(() => markParsing(msgDisplay, character ?? '', idx, renderRevision))

    $effect(() => {
        if(shouldRenderRawStreaming){
            return
        }

        const currentParsing = markParsingResult
        let cancelled = false

        currentParsing.then(async () => {
            await tick()
            if(!cancelled){
                checkImg()
            }
        })

        return () => {
            cancelled = true
        }
    })
</script>

{#if shouldRenderRawStreaming}
    <span class="whitespace-pre-wrap">{rawStreamingText}</span>
{:else}
    {#await markParsingResult}
        {@html addMetadataToElement(trimMarkdown(lastParsed), modelShortName)}
    {:then md}
        {@html addMetadataToElement(trimMarkdown(md), modelShortName)}
    {/await}
{/if}
