<script lang="ts">
    import { browser } from "$app/environment";
    import { getConfirmationPopupContext, getJikuErrorsContext } from "$lib/utils/context.svelte.js";
    import { default_texthooker_options, setTextHookerOptionsContext, type TextHookerOptions } from "./context.js";
    import TopBar from "./TopBar.svelte";
    import OptionPanel from "./OptionPanel.svelte";
	import VirtualList from "$lib/components/VirtualList.svelte";
	import { onMount, tick, untrack } from "svelte";


    let confirmation_popup = getConfirmationPopupContext();

    interface Line {
        id: number,
        text: string,
    }

    let new_lines: Line[] = $state([]);
    let presets: string[] = $state(["Default"]);

    let preset_name: string = $state("");

    if (browser) {
        let preset_stored = localStorage.getItem("selected_preset");
        preset_name = (preset_stored && presets.includes(preset_stored))? preset_stored: "Default";
        fetchLines(preset_name);

        let presets_list_stored = localStorage.getItem("presets");
        if (presets_list_stored) {
            presets = JSON.parse(presets_list_stored);
        }
    }

    $effect(() => {
        if (preset_name) {
            localStorage.setItem("selected_preset", preset_name);
            const new_options = loadOptions(preset_name);
            untrack(() => {
                Object.assign(options, new_options);
            });
            fetchLines(preset_name);
        }
    });

    $effect(() => {
        // TODO: Test indexedDB instead of localStorage
        if (preset_name) {
            localStorage.setItem(`texthooker_lines_preset_${preset_name}`, JSON.stringify(new_lines));
        }
    });

    async function fetchLines (name: string) {
        const stored = localStorage.getItem(`texthooker_lines_preset_${name}`)

        if (stored) {
            new_lines = JSON.parse(stored);
        } else {
            new_lines.length = 0;
        }
    }

    let ws: WebSocket | null = null;
    let ws_connected = $state(false);

    let line_counter: number = $derived(new_lines.length);

    function loadOptions(name: string) {
            let stored;

            if (browser) {
                stored = localStorage.getItem(`texthooker_preset_${name}`);
            }
            
            if (stored) {
                let tmp = JSON.parse(stored);

                Object.keys(default_texthooker_options).forEach((key) => {
                    if (tmp[key] === undefined) {
                        tmp[key] = default_texthooker_options[(key as keyof typeof default_texthooker_options)];
                    }
                });

                return <TextHookerOptions> tmp;
            } else {
                return default_texthooker_options;
            }

    }

    // svelte-ignore state_referenced_locally
    let options: TextHookerOptions = $state(loadOptions(preset_name));


    $effect(() => {
        // console.log(options);
        if (preset_name && options.websocket_url) {
            console.log(`update local storage: texthooker_preset_${preset_name}`);
            localStorage.setItem(`texthooker_preset_${preset_name}`, JSON.stringify(options));
        }
    });

    setTextHookerOptionsContext(options);

    let show_options = $state(false);

    const errors = getJikuErrorsContext();

    // svelte-ignore non_reactive_update
    let text_container: HTMLDivElement;


    function isNearBottom(threshold = 100) {
        const rect = text_container.getBoundingClientRect();
        const scrollBottom = text_container.scrollTop + rect.height;
        return scrollBottom + threshold >= text_container.scrollHeight;
    }

    function isNearLeftMost(threshold = 100) {
        const rect = text_container.getBoundingClientRect();
        const scrollRight = text_container.scrollLeft - rect.width;
        // console.log(scrollRight);
        return -scrollRight + threshold >= text_container.scrollWidth;
    }

    let isNearLast = $derived(options.vertical? isNearLeftMost: isNearBottom);

    let getOffsetLength = $derived(
        options.vertical?
        () => { return text_container.offsetWidth; }:
        () => { return text_container.offsetHeight; }
    );

    let getScrollPosition = $derived(
        options.vertical?
        () => { return -text_container.scrollLeft; }:
        () => { return text_container.scrollTop; }
    );

    let getScrollLength = $derived(
        options.vertical?
        () => { return text_container.scrollWidth; }:
        () => { return text_container.scrollHeight; }
    );


    async function deleteLine(line_id: number) {
        const line_idx = new_lines.findIndex((e) => e.id === line_id);
        vlist.deleteItem(line_idx);
    }

    async function clearAllLines() {
        confirmation_popup.text = "Delete all lines?";
        confirmation_popup.onOk = vlist.clearItems;
        confirmation_popup.show = true;
    }

    function addNewLine(new_line: string) {
        let i = new_lines.length-1;
        let repetition = 0;

        while (i >= 0 && new_lines[i].text === new_line) {
            repetition++;
            i--;
        }

        console.log("repetition", repetition);

        if (repetition > options.max_repetition) { return; }


        let tmp: Line = {
            id: (new_lines.length > 0)? new_lines[new_lines.length-1].id + 1: 0,
            text: new_line,
        };
        
        const near_last: boolean = isNearLast();
        new_lines.push(tmp);

        // tick().then(scrollToLast);
        if (near_last) {
            tick().then(scrollToLast);
        }
    }

    function toggleWebSocket() {
        if (!options.websocket_url) {
            errors.push({
                short: "No WebSocket URL set in options"
            });
            return;
        };

        if (ws == null) {
            ws = new WebSocket(options.websocket_url);

            ws.addEventListener('open', () => {
                console.log("websocket opened");
                ws_connected = true;
            });

            ws.addEventListener('close', () => {
                console.log("websocket closed");
                ws_connected = false;
            });

            ws.addEventListener("message", (event) => {
                if (!event.data) { return; }

                let line;
                try {
                    line = JSON.parse(event.data).sentence;
                } catch {
                    line = event.data;
                }
                
                // console.log(line);
                addNewLine(line);
            })

            ws.addEventListener('error', (err) => {
                console.error('WS error: ', err);
                errors.push({
                    short: "WebSocket Error",
                });
                ws = null;
            });

        } else {
            ws.close();
            ws = null;
        }

    }


    // Virtual List
    let vlist: ReturnType<typeof VirtualList>;
    
    let vertical = $derived(options.vertical);
    let guessed_item_size = $derived(options.font_size * options.line_height);

    let start: number = $state(0);
    let end: number = $state(0);
    let buffer = 5;

    function scrollToLast() {
        vlist.scrollToIndex(-1);
        
        setTimeout(() => {
            const scroll_length = getScrollLength();
            const scroll_pos = getScrollPosition();
            const container_offlength = getOffsetLength();

            if ((scroll_pos + container_offlength) >= scroll_length) {
                return;
            }

            const target = options.vertical? 
                {left: -scroll_length} :
                {top: scroll_length} ;

            text_container.scrollTo(target);

        }, 100);
    }

    let sel: HTMLSelectElement;
    let template: HTMLSelectElement;
    let template_opt: HTMLOptionElement;

    function fitSelection() {
        template.style.display = "block";
        // console.log(template.getBoundingClientRect().width);
        template_opt.text = sel.value;
        const new_w = template.getBoundingClientRect().width;
        template.style.display = "none";
        // console.log(template_opt.text);
        // console.log(new_w);
        sel.style.width = `${new_w}px`;
        sel.style.display = "inline";
    }

    onMount(() => {
        fitSelection();
    });

</script>

<TopBar {toggleWebSocket} {ws_connected} {clearAllLines} {scrollToLast} toggleOptions={() => {show_options = !show_options}}/>

{#if show_options}
    <OptionPanel presets={presets} bind:preset_name={preset_name} onoutsideclick={() => {show_options = false}}/>
{/if}

<div class="h-screen flex flex-col">

    <VirtualList
        bind:this={vlist}
        bind:container={text_container}
        bind:start_idx={start}
        bind:end_idx={end}
        items={new_lines}
        {vertical}
        {guessed_item_size}
        {buffer}
        id="texthooker-container"
        class="relative pt-10 pb-6 w-full grow overflow-auto {options.vertical? "vert-rl pl-5 pr-2": ""}"
        style="line-height: {options.line_height};{options.font? `font-family: ${options.font};`: ""}"
    >
        {#snippet render_item(line, _index)}
            <div class="relative flex items-center">
                <p
                    class="my-1 py-1 px-5 whitespace-pre-wrap"
                    style="font-size: {options.font_size}px;"
                >
                    {line.text}
                </p>
                <button onclick={() => {deleteLine(line.id)}} class="  {options.vertical? "mt-auto mb-1" : "ml-auto mr-4"} hover:text-sky-700 active:text-sky-500 hover:cursor-pointer" title="Delete Line">
                    <svg xmlns="http://www.w3.org/2000/svg" class="w-5" viewBox="0 0 15 15">
                        <path fill="currentColor" d="M3.64 2.27L7.5 6.13l3.84-3.84A.92.92 0 0 1 12 2a1 1 0 0 1 1 1a.9.9 0 0 1-.27.66L8.84 7.5l3.89 3.89A.9.9 0 0 1 13 12a1 1 0 0 1-1 1a.92.92 0 0 1-.69-.27L7.5 8.87l-3.85 3.85A.92.92 0 0 1 3 13a1 1 0 0 1-1-1a.9.9 0 0 1 .27-.66L6.16 7.5L2.27 3.61A.9.9 0 0 1 2 3a1 1 0 0 1 1-1c.24.003.47.1.64.27" />
                    </svg>
                </button>
            </div>
        {/snippet}
    </VirtualList>

    <div class="grow-0 shrink-0 w-full bottom-0 pt-0.5 pb-1 flex items-center justify-between gap-4 text-xs text-neutral-500 border-t border-neutral-700 bg-neutral-800 z-10">   
        <div class="relative left-[50%] translate-x-[-50%] flex items-center justify-between gap-4 px-2">
            <label for="preset">Preset:</label>
            <select bind:this={sel} id="preset" bind:value={preset_name}
                class="max-w-60"
                style="display:none;"
                onchange={(event) => {
                    fitSelection();
                    preset_name = (event.target as HTMLSelectElement).value;
                    // window.location.href = `?preset=${new_preset}`;
                }}
            >
                {#each presets as preset (preset)}
                    <option value={preset}>{preset}</option>
                {/each}
            </select>
        </div>
    
        <span class="mr-4">Lines: {start+1}-{end}/{line_counter}</span>
    </div>

</div>

<select bind:this={template} id="hidden-sel" class="none" style="display:none;">
    <option bind:this={template_opt} id="hidden-sel-option"></option>
</select>


<style>
    .shimmer {
        background: linear-gradient(to right, #3e3e3e 30%,#DCDCDC 50%,#3e3e3e 70%);
        background-size: 400%;
        animation: shimmer 1.5s infinite linear;
    }

    @keyframes shimmer {
	0% {
		background-position: 100%;
	}
	100% {
		background-position: 0%;
	}
}
</style>

