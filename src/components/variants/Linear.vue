<template>
    <div ref="root" :style="`${unique ? '' : `height:calc(100vh - 20rem); width: 100%`}; background: white;`" v-html="lin" />
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref, watch, nextTick } from "vue";
import { linear } from "@/lib/graphieros";

const props = defineProps({
    class: String,
    sequence: String,
    colors: String,
    size: String,
    glyphWidth: String,
    strokeWidth: String,
    activeGlyphInstanceId: String,
    unique: Boolean
});

const emit = defineEmits(["glyph-enter", "glyph-leave"]);

const root = ref(null);

const lin = computed(() =>
    linear({
        class: props.class,
        sequence: props.sequence,
        colors: props.colors.split(","),
        size: props.size,
        glyphWidth: props.glyphWidth,
        width: props.strokeWidth,
        unique: props.unique
    })
);

function applyActiveClass() {
    const el = root.value;
    if (!el) return;

    const paths = el.querySelectorAll("path.glyph");
    paths.forEach((path) => {
        const instanceId = path.dataset?.glyphInstance;
        const isActive = props.activeGlyphInstanceId && instanceId === props.activeGlyphInstanceId;
        path.classList.toggle("is-active-glyph", Boolean(isActive));
    });

    const hits = el.querySelectorAll("path.hit-area");
    hits.forEach((hit) => {
        const instanceId = hit.dataset?.glyphInstance;
        const isActive = props.activeGlyphInstanceId && instanceId === props.activeGlyphInstanceId;
        hit.classList.toggle("is-active-glyph", Boolean(isActive));
    });
}

function onPointerOver(event) {
    const target = event.target;
    if (!target || target.tagName !== "path") return;

    const instanceId = target.dataset?.glyphInstance;
    if (!instanceId) return;

    emit("glyph-enter", instanceId);
}

function onPointerOut(event) {
    const target = event.target;
    if (!target || target.tagName !== "path") return;

    if (!target.dataset?.glyphInstance) return;
    emit("glyph-leave");
}

onMounted(async () => {
    await nextTick();
    const el = root.value;
    if (!el) return;

    el.addEventListener("pointerover", onPointerOver, { passive: true });
    el.addEventListener("pointerout", onPointerOut, { passive: true });

    applyActiveClass();
});

onUnmounted(() => {
    const el = root.value;
    if (!el) return;

    el.removeEventListener("pointerover", onPointerOver);
    el.removeEventListener("pointerout", onPointerOut);
});

watch(
    () => [props.activeGlyphInstanceId, props.sequence],
    async () => {
        await nextTick();
        applyActiveClass();
    }
);
</script>

<style scoped>
:deep(path.glyph.is-active-glyph) {
    filter: brightness(1.2);
    stroke-width: 12;
    stroke: #bf6d4d;
}
:deep(path) {
    transition: all 0.2s;
}
:deep(path.hit-area) {
    transition: fill 0.2s;
}
:deep(path.hit-area.is-active-glyph) {
    fill:#4b576020;
}
svg {
    width: 100%;
}
</style>
