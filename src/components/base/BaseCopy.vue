<script setup>
import { computed, unref } from "vue";
import { VueUiIcon } from "vue-data-ui/vue-ui-icon";
import "vue-data-ui/style.css";

const props = defineProps({
    target: {
        type: [Object, HTMLElement],
        required: true
    },
    filename: {
        type: String,
        default: "graphieros"
    },
    pngScale: {
        type: Number,
        default: 2
    }
});

const targetElement = computed(() => {
    const resolved = unref(props.target);
    return resolved || null;
});

function findSvg() {
    const root = targetElement.value;
    if (!root) return null;

    const tag = root.tagName ? root.tagName.toLowerCase() : "";
    if (tag === "svg") return root;

    return root.querySelector("svg");
}

function downloadBlob(blob, filename) {
    const url = URL.createObjectURL(blob);

    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = filename;
    document.body.appendChild(anchor);
    anchor.click();
    anchor.remove();

    URL.revokeObjectURL(url);
}

function serializeSvg(svgElement) {
    const clone = svgElement.cloneNode(true);

    if (!clone.getAttribute("xmlns")) clone.setAttribute("xmlns", "http://www.w3.org/2000/svg");
    if (!clone.getAttribute("xmlns:xlink")) clone.setAttribute("xmlns:xlink", "http://www.w3.org/1999/xlink");

    const serializer = new XMLSerializer();
    let svgText = serializer.serializeToString(clone);

    if (!svgText.startsWith("<?xml")) {
        svgText = `<?xml version="1.0" encoding="UTF-8"?>\n${svgText}`;
    }

    return svgText;
}

async function handleDownloadSvg() {
    const svgElement = findSvg();
    if (!svgElement) return;

    const svgText = serializeSvg(svgElement);
    const blob = new Blob([svgText], { type: "image/svg+xml;charset=utf-8" });
    downloadBlob(blob, `${props.filename}.svg`);
}

function getExportSize(svgElement) {
    const vb = svgElement.viewBox && svgElement.viewBox.baseVal ? svgElement.viewBox.baseVal : null;

    if (vb && vb.width > 0 && vb.height > 0) {
        return {
            width: Math.round(vb.width),
            height: Math.round(vb.height)
        };
    }

    const rect = svgElement.getBoundingClientRect();
    return {
        width: Math.max(1, Math.round(rect.width)),
        height: Math.max(1, Math.round(rect.height))
    };
}

async function handleDownloadPng() {
    const svgElement = findSvg();
    if (!svgElement) return;

    const { width, height } = getExportSize(svgElement);

    const svgText = serializeSvg(svgElement, width, height);
    const svgBlob = new Blob([svgText], { type: "image/svg+xml;charset=utf-8" });
    const svgUrl = URL.createObjectURL(svgBlob);

    try {
        const image = new Image();
        image.decoding = "async";

        const loaded = new Promise((resolve, reject) => {
            image.onload = resolve;
            image.onerror = reject;
        });

        image.src = svgUrl;
        await loaded;

        const scale = Math.max(1, props.pngScale);

        const canvas = document.createElement("canvas");
        canvas.width = Math.round(width * scale);
        canvas.height = Math.round(height * scale);

        const context = canvas.getContext("2d");
        if (!context) return;

        context.setTransform(scale, 0, 0, scale, 0, 0);
        context.imageSmoothingEnabled = true;
        context.fillStyle = "#ffffff";
        context.fillRect(0, 0, width, height);
        context.drawImage(image, 0, 0, width, height);

        const pngBlob = await new Promise((resolve) =>
            canvas.toBlob(resolve, "image/png")
        );
        if (!pngBlob) return;

        downloadBlob(pngBlob, `${props.filename}.png`);
    } finally {
        URL.revokeObjectURL(svgUrl);
    }
}
</script>

<template>
    <div class="base-copy">
        <button type="button" class="base-copy-button" @click="handleDownloadSvg">
            <VueUiIcon name="fileSvg" stroke="#4b5760"/>
        </button>
        <button type="button" class="base-copy-button" @click="handleDownloadPng">
            <VueUiIcon name="filePng" stroke="#4b5760"/>
        </button>
    </div>
</template>

<style scoped>
.base-copy {
    display: flex;
    align-items: center;
    position: absolute;
    top: 0.5rem;
    right: 0.5rem;
}

.base-copy-button {
    outline: 1px solid #4b5760;
    background-color: white;
    padding: 6px;
    border-radius: 0px;
    font-size: 12px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.base-copy-button:hover {
    background-color: #ebeced;
}
</style>
