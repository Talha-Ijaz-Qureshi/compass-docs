<script lang="ts">
  import { onMount } from "svelte";
  import { Accordion, AccordionItem } from "carbon-components-svelte";
  import "carbon-components-svelte/css/all.css";
  import Icon from "@iconify/svelte";
  import { ProgressIndicator, ProgressStep } from "carbon-components-svelte";
  import { tocOpen } from "$lib/stores/toc";
	import { currentSource, sources } from "$lib/stores/contentStore";
    import { get } from "svelte/store";

  $: sections = $currentSource[0].sections;
  $: topicTitle = $currentSource[0].topic;
  let activeSection: string = "";
  let activeSubtopicId: string = "";
  let observer: IntersectionObserver | null = null;
  let navElement: HTMLElement;

  $: isOpen = $tocOpen;

  function slugify(text: string): string {
    if (!text) return "";
    return text
      .toLowerCase()
      .trim()
      .replace(/\s+/g, "-")
      .replace(/[^\w-]+/g, "")
      .replace(/--+/g, "-")
      .replace(/^-+/, "")
      .replace(/-+$/, "");
  }

  function handleLinkClick(event: Event, href: string) {
    event.preventDefault();
    const targetId = href.slice(1);
    const targetElement = document.getElementById(targetId);

    if (targetElement) {
      const viewportHeight = window.innerHeight;
      const offset = viewportHeight * 0.1;
      const elementTop =
        targetElement.getBoundingClientRect().top + window.scrollY;
      const adjustedTop = elementTop - offset;

      window.scrollTo({
        top: adjustedTop,
        behavior: "smooth",
      });
    }
  }

  function getCurrentSectionIndex(): number {
    const index = sections.findIndex((section) => section.id === activeSection);

    return index;
  }

  function stopPropagation(event: Event) {
    event.stopPropagation();
  }

  function handleClick() {
    tocOpen.update(state => !state);
  }

  function setupObserver() {
    if (typeof window === 'undefined') return;
    
    if (observer) {
      observer.disconnect();
    }

    const subtopicIds: string[] = [];
    sections.forEach((section) => {
      section.subtopics.forEach((subtopic) => {
        const subtopicSlug = slugify(subtopic.title);
        const subtopicId = `${section.id}-${subtopicSlug}`;
        subtopicIds.push(subtopicId);
      });
    });

    const handleIntersection: IntersectionObserverCallback = (entries) => {
      const intersectingEntries = entries.filter(
        (entry) => entry.isIntersecting,
      );
      if (intersectingEntries.length === 0) return;

      let topmostEntry = intersectingEntries[0];
      for (let i = 1; i < intersectingEntries.length; i++) {
        if (
          intersectingEntries[i].boundingClientRect.top <
          topmostEntry.boundingClientRect.top
        ) {
          topmostEntry = intersectingEntries[i];
        }
      }

      if (topmostEntry.target instanceof HTMLElement) {
        const id = topmostEntry.target.id;
        activeSubtopicId = id;
        const sectionId = id.split("-")[0];
        activeSection = sectionId;
      }
    };

    const observerOptions: IntersectionObserverInit = {
      rootMargin: "0px 0px -80% 0px",
      threshold: 0.0,
    };

    observer = new IntersectionObserver(handleIntersection, observerOptions);

    subtopicIds.forEach((id) => {
      const el = document.getElementById(id);
      if (el) {
        observer?.observe(el);
      }
    });
  }

  $: {
    if ($currentSource) {
      setupObserver();
    }
  }

  onMount(() => {
    setupObserver();
    return () => {
      if (observer) {
        observer.disconnect();
      }
    };
  });
</script>

<nav
  bind:this={navElement}
  class:open={$tocOpen}
  aria-label="Table of contents"
  
>

  <ProgressIndicator
on:click={handleClick}
    spaceEqually
    class="toc-progress"
    vertical
    currentIndex={getCurrentSectionIndex()}
  >
    {#each sections as section, index}
      <ProgressStep
        complete={getCurrentSectionIndex()+1 > index}
        description={section.title}
      />
    {/each}
  </ProgressIndicator>

  <h1>{topicTitle}</h1>
  <Accordion>
    {#each sections as section (section.id)}
      <AccordionItem open={activeSection === section.id}>
        <span slot="title">{section.title}</span>
        <ul>
          {#each section.subtopics as subtopic (subtopic.title)}
            {@const subtopicSlug: string = slugify(subtopic.title)}
            {@const subtopicHref: string = `#${section.id}-${subtopicSlug}`}
            <li>
              <a
  href={subtopicHref}
  on:click={(event) => handleLinkClick(event, subtopicHref)}
  class={activeSubtopicId === `${section.id}-${subtopicSlug}` ? "active-link" : "def"}
>
  {subtopic.title}
</a>            </li>
          {/each}
        </ul>
      </AccordionItem>
    {/each}
  </Accordion>
</nav>

<style>

  @import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:ital,wght@0,100..700;1,100..700&display=swap');
  :global(.toc-progress) {
    position: absolute;
    top: 0;
    right: -1.15em;
    height: 100%;
    width: 2rem;
    padding: 1rem;
    padding-top: 0;
    background-color: transparent;
    z-index: 1000;
    height: 100vh;
  }
  /* 
  :global(.toc-progress svg) {
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    -moz-backdrop-filter: blur(10px);
  } */

.active-link {
  color: #ffffff;
  padding: 0em;
  transition: 250ms;
  background-color: #78aaff80;
  border-radius: 0em;
  line-height: 1.3;
}

.def {
    color: var(--cds-link-primary);

}

  span,
  ul {
    word-wrap: break-word;
    overflow-wrap: break-word;
    white-space: normal;
  }

  h1 {
    font-size: 1.5rem;
    margin-bottom: 0.5em;
    margin-top: 0;
    margin-right: .5em;
    margin-left: .5em;
    font-family: "IBM Plex Serif", serif;
    font-weight: 600;
    color: #78a9ff;
    font-stretch: 100%;
    
  }

  nav {
    padding: 1rem 0;
    border-right: 1px solid var(--cds-border-subtle, #e0e0e0);
    width: 330px;
    position: fixed;
    top: 0;
    left: 0;
    height: 100dvh;
    display: flex;
    align-items: center;
    justify-content: flex-start;
    flex-direction: column;
    transition: transform 200ms cubic-bezier(1, 0, 0.01, 1);
    z-index: 1001;
    backdrop-filter: blur(1em);
    -webkit-backdrop-filter: blur(1em);
    -moz-backdrop-filter: blur(1em);
    transition: transform 200ms cubic-bezier(1, 0, 0.01, 1);
    box-shadow: 2px 0 5px rgba(0, 0, 0, 0.1);
    overflow: visible;
    border-right: 2px solid var(--cds-border-subtle, #e0e0e0);
    transform: translateX(0%);

    -webkit-tap-highlight-color: transparent;
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -khtml-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }
  
  nav * {
    -webkit-tap-highlight-color: transparent;
  }
    nav.open {
      transform: translateX(-100%);
    }

  @media (max-width: 1199px) {

    nav {
      transform: translateX(-100%);
      max-width: 270px;

    }

    nav.open {
      transform: translateX(0);
      max-width: 270px;
    }

    h1 {
      padding: 0 0.5em 0 0.5em;
    }
  }

  ul {
    list-style: none;
    padding-left: 1rem;
    margin-top: 0.5rem;
  }

  li {
    margin: 0.5rem 0;
  }

  a {
    color: var(--cds-link-primary, #0f62fe);
    text-decoration: none;
    font-size: var(--cds-body-compact-01-font-size, 0.875rem);
    transition: 100ms;
  }

  a:hover {
    text-decoration: underline;
    color: var(--cds-link-primary-hover, #0043ce);
  }

  @media (max-width: 1199px) {
    .menu-toggle {
      display: block;
    }
  }
</style>
