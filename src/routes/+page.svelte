<script lang="ts">
    import {
        Content,
        InlineNotification,
        CodeSnippet,
        ToastNotification,
        Search,
    } from "carbon-components-svelte";
    import "carbon-components-svelte/css/all.css";
    import { parseContent } from "$lib/utils/parse";
    import { ProgressIndicator, ProgressStep } from "carbon-components-svelte";
    import type { Section } from "$lib/types/content";
    import { currentSource, sources } from "$lib/stores/contentStore";
    import { ComboBox } from "carbon-components-svelte";
    import { slugifyEnabled, camelCaseEnabled } from "$lib/stores/preferences";
    import pkg from 'lodash';
    const { camelCase } = pkg;

    function slugify(text: string, enabled: boolean = true, useCamelCase: boolean = false): string {
        
        if (!enabled && !useCamelCase) return text;
        
        if (useCamelCase) {
            return camelCase(text);
        }
        
        if (enabled) {
            return text
                .toLowerCase()
                .trim()
                .replace(/\s+/g, "-")
                .replace(/[^\w-]+/g, "")
                .replace(/--+/g, "-")
                .replace(/^-+/, "")
                .replace(/-+$/, "");
        }
        
        return text;
    }
    function parseUrls(text: string): string {
        const urlRegex = /(https?:\/\/[^\s]+)/g;
        return text.replace(
            urlRegex,
            '<a href="$1" target="_blank" rel="noopener noreferrer">$1</a>',
        );
    }

    function sanitizeContent(html: string): string {
        const blackList =
            /<(script|iframe|object|embed|form|input|button|style|link|meta|base|textarea|svg|math|xml)[^>]*>/gi;

        return html
            .replace(blackList, "")
            .replace(
                /<[^>]*([^>]*\s(?:on\w+|style|formaction|href="javascript:|data:|vbscript:))[^>]*>/gi,
                "",
            )
            .replace(
                /javascript:|data:|vbscript:|eval\(|expression\(|document\.|window\.|alert\(|confirm\(|prompt\(/gi,
                "",
            )
            .replace(
                /<(?!\/?(a|b|i|p|br|ul|ol|li|h[1-6]|strong|em)\b)[^>]+>/gi,
                "",
            )
            .replace(/(<[^>]+\s)(class|style|id|name)="[^"]*"/gi, "$1")
            .replace(/href="([^"]*)"/gi, (match, url) => {
                if (url.startsWith("http://") || url.startsWith("https://")) {
                    return `href="${url}"`;
                }
                return "";
            });
    }

    function formatText(text: string): string {
        let formatted = parseUrls(text);
        return sanitizeContent(formatted);
    }

    let items = Object.keys(sources).map((key) => ({
        id: key,
        text: key,
    }));

    let selectedValue = "";

    function handleSourceSelect(event: CustomEvent) {
        const selected = event.detail.selectedId as keyof typeof sources;
        if (selected in sources) {
            $currentSource = sources[selected];
        }
    }

    function handleClear() {
        selectedValue = "";
    }

    $: topicTitle = $currentSource[0].topic;

    function searchFilter(item: { text: string }, value: string): boolean {
        if (!value) return true;
        return item.text.toLowerCase().includes(value.toLowerCase());
    } 
</script>

<div class="header">
    <div id="search" class="searchContainer">
        <ComboBox
            size="sm"
            titleText="Select Documentation Source"
            placeholder="Choose a documentation source..."
            {items}
            bind:value={selectedValue}
            on:select={handleSourceSelect}
            on:focus={handleClear}
            shouldFilterItem={searchFilter}
            invalidText="Erroneous source"
        />
    </div>

    <hr />
    <h1 class="hide">{topicTitle}</h1>

    <!-- <svg viewBox='0 0 250 250' xmlns='http://www.w3.org/2000/svg'>
  <filter id='noiseFilter'>
    <feTurbulence
      type='fractalNoise'
      baseFrequency='3'
      numOctaves='1000'
      stitchTiles='stitch'/>
  </filter>

  <rect width='100%' height='100%' filter='url(#noiseFilter)'/>
</svg> -->
</div>

<Content class="documentation">
    {#each $currentSource[0].sections as section}
        <section id={section.id}>
            <h2>{slugify(section.title, $slugifyEnabled, $camelCaseEnabled)}</h2>
            <p>{@html section.intro}</p>
            {#each section.subtopics as subtopic}
                {@const subtopicSlug: string = slugify(subtopic.title, true, false)}
                {@const subtopicId: string = `${section.id}-${subtopicSlug}`}
                <h3 id={subtopicId}>{slugify(subtopic.title, $slugifyEnabled, $camelCaseEnabled)}</h3>
                {#each subtopic.blocks as block}
                    {#if block.type === "text"}
                        <p style="margin: 1em 0 .2em 0;">
                            {@html formatText(block.value)}
                        </p>
                    {:else if block.type === "code"}
                        <CodeSnippet
                            type={block.codeBlock}
                            
                            showMoreLess={false}
                            code={block.value}
                        />
                    {:else if block.type === "note"}
                        <InlineNotification
                            kind={block.kind}
                            title={block.message}
                            subtitle={block.value}
                            hideCloseButton
                            lowContrast
                        />
                    {/if}
                {/each}
            {/each}
            <hr />
        </section>
    {/each}
</Content>

<style>
    @import url("https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:ital,wght@0,100..700;1,100..700&display=swap");
    
    :global(*) {
        -webkit-tap-highlight-color: transparent;
    }
    
    :global(ol) {
        margin: 0.1em;
        margin-left: 2em;
        line-height: 2;
    }

    .header {
        padding: 1rem 0rem;
        padding-top: 1.2rem;
        overflow: hidden;
        width: 100%;
        box-shadow: 3px 7px 15px #00000042;
        border-radius: 0 0 1em 1em;
        border: 1px solid var(--cds-border-subtle, #e0e0e0);
        border-top: none;
        margin: 0rem 2rem;
        background-image: linear-gradient(to top, #78aaff34, transparent);
    }

    :global(.documentation b) {
        /* background-color: var(--cds-border-subtle); */
        background-color: #78aaff28;
        padding: 0.2rem 0.4rem;
        border-radius: 4px;
        color: var(--cds-link-primary);
        font-family: "IBM Plex Mono", monospace;
        font-size: 0.9rem;
        font-weight: 500;
        display: inline;
        word-break: break-word;
    }
    
    :global(.bx--content) {
        overflow-x: hidden;
        max-width: 100%;
    }
    
    .hide {
        display: block;
    }
    h1 {
        font-size: 5rem;
        margin-left: 2rem;
        margin-bottom: 0rem;
        margin-top: 2rem;
        font-family: "IBM Plex Serif", serif;
        font-weight: 600;
        color: #78a9ff;
        font-stretch: 100%;
        position: relative;
        word-wrap: break-word;
        overflow-wrap: break-word;
        white-space: normal;
        z-index: 1;

        /* text-shadow: */
        /* 1.5px 1.5px 1px var(--cds-text-primary), */
        /* -1.5px -1.5px 1px var(--cds-text-primary),
        1.5px -1.5px 1px var(--cds-text-primary),
        1.5px 1.5px 1px var(--cds-text-primary), */
        /* 1em .5em .1em #78aaff38,
		-1em -.5em .1em #78a9ff38 */
        /* 5px -5px 3em var(--cds-text-primary),
        5px -5px 3em var(--cds-text-primary),

        var(--cds-text-primary) .001em .001em 1em */

        /* text-shadow:
  1px 0px 0 #000,
  -1px 0px 0 #000,
  0px 1px 0 #000,
  0px -1px 0 #000,
  1px 1px 0 #000,
  -1px -1px 0 #000,
  -1px 1px 0 #000,
  1px -1px 0 #000,
  0.5px 0.5px 0 #000,
  -0.5px -0.5px 0 #000,
  -0.5px 0.5px 0 #000,
  0.5px -0.5px 0 #000,
  0.75px 0.75px 0 #000,
  -0.75px -0.75px 0 #000,
  -0.75px 0.75px 0 #000,
  0.75px -0.75px 0 #000; */
    }
    h2 {
        font-size: 2.5rem;
        /* color: #c6c6c6; */
        margin: 2rem 0 1rem;
        font-family: "IBM Plex Serif", serif;
        opacity: 0.9;
        word-wrap: break-word;
        overflow-wrap: break-word;
        white-space: normal;
    }

    h3 {
        font-size: 1.5rem;
        margin: 1.5rem 0 0.5rem;
        opacity: 0.9;
        word-wrap: break-word;
        overflow-wrap: break-word;
        white-space: normal;
    }
    /* Weird gradient appearing in code snippets */
    :global(.bx--snippet--multi .bx--snippet-container pre::after) {
        background-image: none;
    }

    hr {
        border: 0;
        border-top: 1.5px solid var(--cds-border-subtle);
        margin: 2rem 0;
    }
    p {
        line-height: 2;
        word-wrap: break-word;
        overflow-wrap: break-word;
        white-space: normal;
    }

    .searchContainer {
        max-width: 450px;
        padding: 1em 2em;
    }

    .searchContainer {
        opacity: 0.7;
        transition: opacity 0.2s ease;
    }

    .searchContainer:hover {
        opacity: 1;
    }

    @media (max-width: 1199px) {
        .searchContainer {
            padding: 1em 2em;
            margin: 1rem 0.5rem;
        }

        h1 {
            font-size: 3.2rem;
            margin-left: 2rem;
            margin-top: 1rem;
        }
        h2 {
            font-size: 1.9rem;
        }
        h3 {
            font-size: 1.5rem;
        }
        p {
            font-size: 0.95rem;
            line-height: 1.7;
        }
        .hide {
            display: block;
        }

        :global(.documentation) {
            margin: 0 0.5em;
        }
    }

    @media (max-width: 768px) {
        .header {
            width: 82%;
            margin: auto;
        }

        p {
            max-width: 60ch;
        }
    }
</style>
