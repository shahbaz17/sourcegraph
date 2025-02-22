<svelte:options immutable />

<script lang="ts" context="module">
    const BY_LINE_RANKING = 'by-line-number'
    const DEFAULT_CONTEXT_LINES = 1
    const MAX_LINE_MATCHES = 5
    const MAX_ZOEKT_RESULTS = 3
</script>

<script lang="ts">
    import { mdiChevronDown, mdiChevronUp } from '@mdi/js'

    import {
        addLineRangeQueryParameter,
        formatSearchParameters,
        pluralize,
        toPositionOrRangeQueryParameter,
    } from '$lib/common'
    import Icon from '$lib/Icon.svelte'
    import { getFileMatchUrl, type ContentMatch, ZoektRanking, LineRanking } from '$lib/shared'

    import SearchResult from './SearchResult.svelte'
    import { getSearchResultsContext } from './SearchResults.svelte'
    import CodeHostIcon from './CodeHostIcon.svelte'
    import RepoStars from './RepoStars.svelte'
    import { settings } from '$lib/stores'
    import { rankContentMatch } from '$lib/search/results'
    import FileSearchResultHeader from './FileSearchResultHeader.svelte'
    import { fetchFileRangeMatches } from '$lib/search/api/highlighting'
    import CodeExcerpt from './CodeExcerpt.svelte'

    export let result: ContentMatch

    $: contextLines = $settings?.['search.contextLines'] ?? DEFAULT_CONTEXT_LINES
    $: ranking =
        $settings?.experimentalFeatures?.clientSearchResultRanking === BY_LINE_RANKING
            ? new LineRanking(MAX_LINE_MATCHES)
            : new ZoektRanking(MAX_ZOEKT_RESULTS)
    $: ({ expandedMatchGroups, collapsedMatchGroups, collapsible, hiddenMatchesCount } = rankContentMatch(
        result,
        ranking,
        contextLines
    ))
    $: fileURL = getFileMatchUrl(result)

    const searchResultContext = getSearchResultsContext()
    let expanded: boolean = searchResultContext?.isExpanded(result)
    $: searchResultContext.setExpanded(result, expanded)
    $: expandButtonText = expanded
        ? 'Show less'
        : `Show ${hiddenMatchesCount} more ${pluralize('match', hiddenMatchesCount, 'matches')}`
    $: matchesToShow = expanded ? expandedMatchGroups.grouped : collapsedMatchGroups.grouped
    $: matchRanges = matchesToShow.map(group => ({
        startLine: group.startLine,
        endLine: group.endLine,
    }))

    let root: HTMLElement
    let userInteracted = false
    $: if (!expanded && root && userInteracted) {
        setTimeout(() => {
            const reducedMotion = !window.matchMedia('(prefers-reduced-motion: no-preference)').matches
            root.scrollIntoView({ block: 'nearest', behavior: reducedMotion ? 'auto' : 'smooth' })
        }, 0)
    }

    function getMatchURL(startLine: number, endLine: number): string {
        const searchParams = formatSearchParameters(
            addLineRangeQueryParameter(
                // We don't want to preserve the 'q' query parameter.
                // We might have to adjust this if we want to preserve other query parameters.
                new URLSearchParams(),
                toPositionOrRangeQueryParameter({ range: { start: { line: startLine }, end: { line: endLine } } })
            )
        )
        return `${fileURL}?${searchParams}`
    }

    async function fetchHighlightedFileMatchLineRanges(startLine: number, endLine: number) {
        const highlightedGroups = await fetchFileRangeMatches({ result, ranges: matchRanges })
        return highlightedGroups[
            matchesToShow.findIndex(group => group.startLine === startLine && group.endLine === endLine)
        ]
    }
</script>

<SearchResult>
    <CodeHostIcon slot="icon" repository={result.repository} />
    <FileSearchResultHeader slot="title" {result} />
    <svelte:fragment slot="info">
        {#if result.repoStars}
            <RepoStars repoStars={result.repoStars} />
        {/if}
    </svelte:fragment>

    <div bind:this={root} class="matches">
        {#each matchesToShow as group}
            <div class="code">
                <a href={getMatchURL(group.startLine + 1, group.endLine)}>
                    <CodeExcerpt
                        startLine={group.startLine}
                        endLine={group.endLine}
                        fetchHighlightedFileRangeLines={async (...args) =>
                            group.blobLines ? group.blobLines : fetchHighlightedFileMatchLineRanges(...args)}
                        matches={group.matches}
                    />
                </a>
            </div>
        {/each}
    </div>
    {#if collapsible}
        <button
            type="button"
            on:click={() => {
                expanded = !expanded
                userInteracted = true
            }}
            class:expanded
        >
            <Icon svgPath={expanded ? mdiChevronUp : mdiChevronDown} inline aria-hidden="true" />
            <span>{expandButtonText}</span>
        </button>
    {/if}
</SearchResult>

<style lang="scss">
    button {
        width: 100%;
        text-align: left;
        border: none;
        padding: 0.25rem 0.5rem;
        background-color: var(--border-color);
        border-radius: 0 0 var(--border-radius) var(--border-radius);
        color: var(--collapse-results-color);
        cursor: pointer;

        &.expanded {
            position: sticky;
            bottom: 0;
        }
    }

    .matches {
        border-radius: var(--border-radius);
        border: 1px solid var(--border-color);
        background-color: var(--code-bg);
    }

    .code {
        &:not(:first-child) {
            border-top: 1px solid var(--border-color);
        }

        a {
            text-decoration: none;
            color: inherit;
        }
    }
</style>
