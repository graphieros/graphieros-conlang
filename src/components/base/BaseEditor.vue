<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from "vue";
import Linear from "../variants/Linear.vue";
import { graphieros_dictionnary } from "@/lib/graphieros";
import BaseCopy from "./BaseCopy.vue";

const userInput = ref(" mea-ka-tsi | le-kto-pa-tsi");
const linearSequence = ref("");
const selectedLang = ref("en");
const textarea = ref(null);
const linearRoot = ref(null);

const toggleClass = ref("toggle-right");

const tokens = ref([]);
const activeGlyphInstanceId = ref(null);

// suggestions
const isSuggestionsOpen = ref(false);
const activeSuggestionIndex = ref(0);
const suggestionsLimit = 8;

const suggestionsRoot = ref(null);

const dictionaryIndex = computed(() => {
    return graphieros_dictionnary.map((entry) => {
        const raw = entry.name && entry.name.startsWith("_") ? entry.name.slice(1) : entry.name;

        return {
            name: entry.name,
            raw,
            fr: entry.fr,
            en: entry.en,
            type: entry.type,
            description: entry.description
        };
    });
});

onMounted(() => {
    textarea.value?.focus();
    buildOutput();
    document.addEventListener("pointerdown", handleGlobalPointerDown, true);
});

onUnmounted(() => {
    document.removeEventListener("pointerdown", handleGlobalPointerDown, true);
});

function closeSuggestions() {
    isSuggestionsOpen.value = false;
}

function handleGlobalPointerDown(event) {
    if (!isSuggestionsOpen.value) return;

    const root = suggestionsRoot.value;
    const text = textarea.value;

    const target = event.target;

    const clickedInsideSuggestions = root && root.contains(target);
    const clickedTextarea = text && text.contains(target);

    if (!clickedInsideSuggestions && !clickedTextarea) {
        closeSuggestions();
    }
}

function handleEnter(event) {
    if (event.key === "Enter") {
        if (isSuggestionsOpen.value && suggestions.value.length > 0) {
            event.preventDefault();
            const item = suggestions.value[activeSuggestionIndex.value];
            if (item) applySuggestion(item);
            return;
        }

        event.preventDefault();
        userInput.value += " | ";
        linearSequence.value += " | ";
        buildOutput();
    }
}

function buildOutput() {
    const normalized = (userInput.value || "").toLowerCase().replaceAll("/", " ");
    linearSequence.value = normalized === "" ? "ne" : normalized;

    tokens.value = [];
    activeGlyphInstanceId.value = null;

    if (normalized === "") return;

    const lines = normalized.split(" | ");
    let glyphInstanceCounter = 0;

    lines.forEach((line, lineIndex) => {
        const words = line.split(" ").filter(Boolean);

        words.forEach((word) => {
            const parts = word.split("-").filter(Boolean);

            parts.forEach((part, partIndex) => {
                const glyphName = `_${part}`;
                const entry = graphieros_dictionnary.find((item) => item.name === glyphName);
                if (!entry) return;

                const instanceId = `l${lineIndex}-g${glyphInstanceCounter++}`;

                tokens.value.push({
                    instanceId,
                    glyphName,
                    labelFr: entry.fr,
                    labelEn: entry.en,
                    isSection: partIndex > 0
                });
            });
        });
    });
}

function deleteInput() {
    userInput.value = "";
    linearSequence.value = "ne";
    tokens.value = [];
    activeGlyphInstanceId.value = null;
    isSuggestionsOpen.value = false;
}

const showTranslation = computed(() => {
    if (userInput.value === "") return "display: none;";
    return `
        display: block;
        position: relative;
        width: 100%;
        max-width: 600px;
        box-sizing: border-box;
        padding: 24px;
        font-family: var(--logo);
        color: RGB(var(--c3));
        margin-top: 124px;
        border-radius: 15px;
        box-shadow: 0 10px 20px -10px RGBA(var(--c0), 0.5);
        font-size: 0.8em;
    `;
});

function getCaretIndex() {
    return textarea.value ? textarea.value.selectionStart : (userInput.value || "").length;
}

function getCurrentTokenContext() {
    const text = userInput.value || "";
    const caret = getCaretIndex();

    const before = text.slice(0, caret);
    const after = text.slice(caret);

    const lastSeparatorMatch = before.match(/.*[ \-\/|]\s*/);
    const tokenStart = lastSeparatorMatch ? lastSeparatorMatch[0].length : 0;

    const token = before.slice(tokenStart);

    const nextSeparatorMatch = after.match(/[ \-\/|]/);
    const tokenEnd = nextSeparatorMatch ? caret + nextSeparatorMatch.index : text.length;

    return { tokenStart, tokenEnd, token };
}

const query = computed(() => {
    const { token } = getCurrentTokenContext();
    const value = (token || "").trim().toLowerCase();

    if (value === "|" || value === "-" || value === "/") return "";
    return value;
});

function normalizeForSearch(value) {
    return String(value || "")
        .toLowerCase()
        .normalize("NFD")
        .replace(/[\u0300-\u036f]/g, "");
}

const suggestions = computed(() => {
    const q = normalizeForSearch(query.value);
    if (!q) return [];

    const prefixMatches = [];
    const containsMatches = [];

    for (const item of dictionaryIndex.value) {
        const rawLower = normalizeForSearch(item.raw);
        const enLower = normalizeForSearch(item.en);
        const frLower = normalizeForSearch(item.fr);

        if (rawLower.startsWith(q)) {
            prefixMatches.push(item);
            continue;
        }

        if (rawLower.includes(q) || enLower.includes(q) || frLower.includes(q)) {
            containsMatches.push(item);
        }
    }

    return [...prefixMatches, ...containsMatches].slice(0, suggestionsLimit);
});

watch([query, () => userInput.value], () => {
    activeSuggestionIndex.value = 0;
    isSuggestionsOpen.value = suggestions.value.length > 0;
});

function applySuggestion(item) {
    const { tokenStart, tokenEnd } = getCurrentTokenContext();
    const text = userInput.value || "";

    const before = text.slice(0, tokenStart);
    const after = text.slice(tokenEnd);

    const inserted = item.raw;

    userInput.value = `${before}${inserted}${after}`;

    buildOutput();

    isSuggestionsOpen.value = false;

    requestAnimationFrame(() => {
        if (!textarea.value) return;
        const newCaret = tokenStart + inserted.length;
        textarea.value.setSelectionRange(newCaret, newCaret);
        textarea.value.focus();
    });
}

function handleTextareaKeydown(event) {
    if (event.key === "Enter") {
        handleEnter(event);
        return;
    }

    if (event.key === "Escape") {
        event.preventDefault();
        closeSuggestions();
        return;
    }

    if (!isSuggestionsOpen.value || suggestions.value.length === 0) {
        return;
    }

    if (event.key === "ArrowDown") {
        event.preventDefault();
        activeSuggestionIndex.value = (activeSuggestionIndex.value + 1) % suggestions.value.length;
        return;
    }

    if (event.key === "ArrowUp") {
        event.preventDefault();
        activeSuggestionIndex.value =
            (activeSuggestionIndex.value - 1 + suggestions.value.length) % suggestions.value.length;
        return;
    }

    if (event.key === "Tab") {
        event.preventDefault();
        const item = suggestions.value[activeSuggestionIndex.value];
        if (item) applySuggestion(item);
        return;
    }
}

function getHighlightedText(text, queryValue) {
    const t = String(text || "");
    const q = String(queryValue || "").trim();
    if (!q) return t;

    const safe = q.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
    const regex = new RegExp(safe, "ig");
    return t.replace(regex, (match) => `<mark>${match}</mark>`);
}

const suggestionsWithHighlights = computed(() => {
    const q = query.value;
    if (!q) return suggestions.value;

    const normalizedQuery = normalizeForSearch(q);

    return suggestions.value.map((item) => {
        const rawMatches = normalizeForSearch(item.raw).includes(normalizedQuery);
        const enMatches = normalizeForSearch(item.en).includes(normalizedQuery);
        const frMatches = normalizeForSearch(item.fr).includes(normalizedQuery);

        return {
            ...item,
            rawHighlighted: rawMatches ? getHighlightedText(item.raw, q) : item.raw,
            enHighlighted: enMatches ? getHighlightedText(item.en, q) : item.en,
            frHighlighted: frMatches ? getHighlightedText(item.fr, q) : item.fr
        };
    });
});
</script>

<template>
    <div class="flex w-full min-h-screen max-w-350" style="margin: 0 auto">
        <div class="editor-body w-full flex justify-center">
            <div class="editor-playground flex flex-col w-full">
                <div class="relative w-full">
                    <label>
                        <span class="text-[#4b5760]">
                            Type to view suggestions
                        </span>
                        <span class="text-[#a4aaae]">|</span>
                        <span class="text-[#6e787f]">
                            Saisissez du texte pour voir des suggestions
                        </span>
                        <textarea
                            class="bg-white p-1 w-full min-w-full min-h-30 focus:outline-2 focus:outline-[#4b5760]"
                            ref="textarea"
                            v-model="userInput"
                            @input="buildOutput"
                            @keydown="handleTextareaKeydown"
                            placeholder="input graphieros phonology here, for example: kli-keo mea-kadwa / kio-tew-ma !"
                        />
                    </label>

                    <div
                        v-if="isSuggestionsOpen && suggestionsWithHighlights.length > 0"
                        ref="suggestionsRoot"
                        class="absolute left-0 right-0 mt-2 bg-white border border-black rounded-md shadow-lg z-50 overflow-hidden"
                    >
                        <div
                            v-for="(item, index) in suggestionsWithHighlights"
                            :key="item.name"
                            class="px-3 py-2 cursor-pointer select-none"
                            :class="index === activeSuggestionIndex ? 'bg-gray-200' : ''"
                            @mousedown.prevent="applySuggestion(item)"
                            @mouseenter="activeSuggestionIndex = index"
                        >
                            <div class="suggestion-row">
                                <div class="suggestion-preview">
                                    <Linear
                                        :sequence="item.raw"
                                        colors="75,87,96"
                                        glyphWidth="100"
                                        strokeWidth="12"
                                        unique
                                    />
                                </div>

                                <div class="suggestion-meta">
                                    <div class="suggestion-top">
                                        <span class="font-mono" v-html="item.rawHighlighted" />
                                        <span class="text-xs opacity-70">{{ item.type }}</span>
                                    </div>

                                    <div class="text-xs opacity-80">
                                        <span v-html="item.enHighlighted" />
                                        <span class="opacity-60"> · </span>
                                        <span v-html="item.frHighlighted" />
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                
                <div class="editor-output-active w-full relative">
                    <div class="flex items-center justify-end mb-2">
                        <BaseCopy v-if="linearRoot" :target="linearRoot" filename="graphieros-linear" :pngScale="2" />
                    </div>
                    <div  ref="linearRoot">
                        <Linear
                            :sequence="linearSequence"
                            colors="75,87,96"
                            :activeGlyphInstanceId="activeGlyphInstanceId"
                            @glyph-enter="activeGlyphInstanceId = $event"
                            @glyph-leave="activeGlyphInstanceId = null"
                        />
                    </div>
                </div>
            </div>
        </div>

        <div class="translation-hub w-full flex justify-center">
            <div :style="showTranslation">
                <div class="pb-2">
                    <span class="text-[#4b5760]">
                            Litteral translation
                        </span>
                        <span class="text-[#a4aaae]">|</span>
                        <span class="text-[#6e787f]">
                            Traduction littérale
                        </span>
                </div>
                <span class="explain">
                    <div
                        v-for="token in tokens"
                        :key="token.instanceId"
                        class="translation-token flex flex-col"
                        :class="[
                            token.isSection ? 'token-section' : '',
                            token.instanceId === activeGlyphInstanceId ? 'is-active' : ''
                        ]"
                        @mouseenter="activeGlyphInstanceId = token.instanceId"
                        @mouseleave="activeGlyphInstanceId = null"
                    >
                        <span class="text-[#4b5760]">{{ token.labelEn }}</span>
                        <span class="text-[#6e787f]">{{ token.labelFr }}</span>
                    </div>
                </span>
            </div>
        </div>
    </div>
</template>

<style lang="scss">
.translation-token {
    background: white;
    padding: 0.25rem 0.5rem;
    position: relative;
    border-radius: 0.25rem;
    color: white;
    transition: transform 120ms linear, filter 120ms linear, outline 120ms linear;
    user-select: none;
    line-height: 0.8rem;
    border-radius: 1rem;
    border: 2px solid white;
}

.translation-token.is-active {
    outline: 2px solid currentColor;
    transform: translateY(-1px);
    border: 2px solid #bf6d4d;
}

.token-section::before {
    content: "";
    position: absolute;
    top: 50%;
    right: 100%;
    transform: translateY(-50%);
    height: 0.5rem;
    width: 0.5rem;
    background: white;
}

.explain {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    gap: 0.5rem;
}

path {
    transition: all 0.2s !important;
}

mark {
    background: rgba(255, 235, 59, 0.6);
    padding: 0 2px;
    border-radius: 2px;
}

.suggestion-row {
    display: flex;
    align-items: center;
    gap: 10px;
}

.suggestion-preview {
    width: 54px;
    height: 54px;
    border: 1px solid rgba(0, 0, 0, 0.25);
    border-radius: 8px;
    background: white;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
}

.suggestion-preview :deep(svg) {
    width: 44px;
    height: 44px;
}

.suggestion-meta {
    flex: 1;
    min-width: 0;
}

.suggestion-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
}
</style>
