<template>
    <v-container>
        <v-file-input label="Bild hochladen" @change="handleFileUpload" accept="image/*" />
        <div v-if="imageSrc" class="image-container">
            <img ref="image" :src="imageSrc" @load="onImageLoad" />
            <div class="overlay" :style="{top: rect.top + 'px', left: rect.left + 'px', width: rect.width + 'px', height: rect.height + 'px'}"></div>
        </div>
        <div :style="{position: 'fixed', top: '10px', left: '10px', width: '200px', height: 'auto', background: 'red'}">
            <div class="d-flex flex-column ga-4">
                <v-slider v-model="rect.width" :max="maxWidth" label="Breite" />
                <v-slider v-model="rect.height" :max="maxHeight" label="Höhe" />
                <br />
                <div>w: {{ rect.width }}, h: {{ rect.height }}, t: {{ rect.top }}, l: {{ rect.left }}</div>
                <v-btn class="mx-4" flat size="small" @click="move('up')">Nach oben</v-btn>
                <v-btn class="mx-4" flat size="small" @click="move('down')">Nach unten</v-btn>
                <v-btn class="mx-4" flat size="small" @click="move('left')">Nach links</v-btn>
                <v-btn class="mx-4" flat size="small" @click="move('right')">Nach rechts</v-btn>

                <v-btn @click="exportSelection">Ausschnitt exportieren</v-btn>
                <br />
                <v-slider v-model="threshold1" min="0" max="255" step="1" label="Threshold1" />
                <v-slider v-model="threshold2" min="0" max="255" step="1" label="Threshold2" />
                <v-slider v-model="cannyThreshold1" min="0" max="255" step="1" label="Canny Threshold1" />
                <v-slider v-model="cannyThreshold2" min="0" max="255" step="1" label="Canny Threshold2" />
                <div>t1: {{ threshold1 }}, t2:{{ threshold2 }}, c1:{{ cannyThreshold1 }}, c2: {{ threshold2 }}</div>
                <v-btn @click="detectMiddleColumn">Anwenden</v-btn>
                <br />
            </div>
        </div>
        <canvas
            ref="canvasOutput"
            @mousedown="startSelection($event)"
            @mousemove="drawSelection($event)"
            @mouseup="endSelection($event)"
            @touchstart="startSelection($event)"
            @touchmove="drawSelection($event)"
            @touchend="endSelection($event)"
        ></canvas>
    </v-container>
</template>

<script setup>
import cv from '@techstark/opencv-js';
import Tesseract, {PSM} from 'tesseract.js';
import {watch} from 'vue';
import {watchEffect} from 'vue';

import {onUnmounted} from 'vue';
import {onMounted} from 'vue';
import {ref} from 'vue';

const imageSrc = ref(null);
const image = ref(null);
const canvasOutput = ref(null);
const canvasImage = ref(null);
const canvasSelection = ref({
    x1: null,
    y1: null,
    x2: null,
    y2: null,
    isDrawing: false,
});

const threshold1 = ref(127); // hell dunkel
const threshold2 = ref(222); // hintergrund grau
const cannyThreshold1 = ref(50);
const cannyThreshold2 = ref(100);

const rect = ref({top: 40, left: 60, width: 80, height: 38});
// Maximal zulässige Breite = bildbreite
const maxWidth = ref(image.value ? image.value.width : 1800);
// Maximal zulässige Höhe = bildhöhe
const maxHeight = ref(image.value ? image.value.height : 1800);

const step = 1;

function handleFileUpload(event) {
    const file = event.target.files[0];
    imageSrc.value = URL.createObjectURL(file);
    // Initialisiere die Rechteck-Daten
}

function move(direction) {
    switch (direction) {
        case 'up':
            rect.value.top = Math.max(0, rect.value.top - step);
            break;
        case 'down':
            rect.value.top = Math.min(maxHeight.value - rect.value.height, rect.value.top + step);
            break;
        case 'left':
            rect.value.left = Math.max(0, rect.value.left - step);
            break;
        case 'right':
            rect.value.left = Math.min(maxWidth.value - rect.value.width, rect.value.left + step);
            break;
    }
}

function handleKeyDown(event) {
    switch (event.key) {
        case 'ArrowUp':
            rect.value.top = Math.max(0, rect.value.top - step);
            break;
        case 'ArrowDown':
            rect.value.top = Math.min(maxHeight.value - rect.value.height, rect.value.top + step);
            break;
        case 'ArrowLeft':
            rect.value.left = Math.max(0, rect.value.left - step);
            break;
        case 'ArrowRight':
            rect.value.left = Math.min(maxWidth.value - rect.value.width, rect.value.left + step);
            break;
    }
}

function detectMiddleColumn() {
    if (!image.value) return;

    const src = cv.imread(image.value);

    console.log(src.cols);
    // const halfWidth = src.cols / 3;
    // console.log(halfWidth);
    // Schneiden Sie das Bild auf die linke Hälfte, falls gewünscht
    const tempRect = new cv.Rect(0, 0, src.cols, src.rows);

    const gray = new cv.Mat();
    cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY, 0);

    // Hier könnten weitere Verarbeitungsschritte hinzukommen, wie Thresholding oder Canny Edge Detection
    const thresh = new cv.Mat();
    cv.threshold(gray, thresh, threshold1.value, threshold2.value, cv.THRESH_BINARY);

    // cv.Canny(thresh, thresh, cannyThreshold1.value, cannyThreshold2.value, 3, false);
    cv.imshow(canvasOutput.value, thresh);

    const boundingRect = cv.boundingRect(thresh);
    rect.value = {
        top: boundingRect.y,
        left: boundingRect.x,
        width: boundingRect.width,
        height: boundingRect.height,
    };

    // Finden Sie Konturen und berechnen Sie Bounding Boxes
    /*
    const contours = new cv.MatVector();
    const hierarchy = new cv.Mat();
    cv.findContours(gray, contours, hierarchy, cv.RETR_CCOMP, cv.CHAIN_APPROX_SIMPLE);
    let maxArea = 0;
    let maxContourIdx = -1;
    for (let i = 0; i < contours.size(); ++i) {
        const cnt = contours.get(i);
        const area = cv.contourArea(cnt);
        if (area > maxArea) {
            maxArea = area;
            maxContourIdx = i;
        }
    }

    if (maxContourIdx !== -1) {
        const boundingRect = cv.boundingRect(contours.get(maxContourIdx));
        rect.value = {
            top: boundingRect.y,
            left: boundingRect.x,
            width: boundingRect.width,
            height: boundingRect.height,
        };
    }
	*/

    // Bereinigung
    src.delete();
    gray.delete();
    // contours.delete();
    // hierarchy.delete();
    // halfImg.delete();
}

// Diese Funktion wird aufgerufen, wenn das Bild vollständig geladen ist
function onImageLoad() {
    // Stellen Sie sicher, dass OpenCV bereit ist, bevor Sie versuchen, das Bild zu verarbeiten
    detectMiddleColumn();
}

watch(threshold1, async (newTresh, oldTresh) => {
    detectMiddleColumn();
});
watch(threshold2, async (newTresh, oldTresh) => {
    detectMiddleColumn();
});
watch(cannyThreshold1, async (newTresh, oldTresh) => {
    detectMiddleColumn();
});
watch(cannyThreshold2, async (newTresh, oldTresh) => {
    detectMiddleColumn();
});

onMounted(() => {
    window.addEventListener('keydown', handleKeyDown);
});

onUnmounted(() => {
    window.removeEventListener('keydown', handleKeyDown);
});

function exportSelection() {
    const imageName = prompt('Bitte geben Sie einen Namen für die herunterzuladende Datei ein:', 'exported-image');
    if (imageName === null || imageName.trim() === '') {
        alert('Der Dateiname darf nicht leer sein. Vorgang abgebrochen.');
        return;
    }
    // Implementiere Logik zum Exportieren des ausgewählten Bereichs
    const image = new Image();
    image.src = imageSrc.value;

    image.onload = () => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');

        // Setzen Sie die Größe des Canvas auf die Größe des ausgewählten Bereichs
        canvas.width = rect.value.width;
        canvas.height = rect.value.height;

        // Zeichnen Sie den ausgewählten Bereich des Bildes auf das Canvas
        ctx.drawImage(image, rect.value.left, rect.value.top, rect.value.width, rect.value.height, 0, 0, rect.value.width, rect.value.height);

        // Konvertieren Sie das Canvas in ein Bild und speichern Sie es
        canvas.toBlob(blob => {
            const link = document.createElement('a');
            link.href = URL.createObjectURL(blob);
            link.download = imageName + '.png'; // Name der heruntergeladenen Datei
            link.click();
            URL.revokeObjectURL(link.href); // Bereinigen
        }, 'image/png');
    };
}

const startSelection = event => {
    event.preventDefault();

    if (!imageSrc.value || canvasSelection.value.isDrawing) return;

    canvasImage.value = new Image();
    canvasImage.value.src = canvasOutput.value.toDataURL('image/png');
    const clientX = event.touches ? event.touches[0].clientX : event.clientX;
    const clientY = event.touches ? event.touches[0].clientY : event.clientY;

    const rect = canvasOutput.value.getBoundingClientRect();
    canvasSelection.value = {
        x1: clientX - rect.left,
        y1: clientY - rect.top,
        x2: clientX - rect.left,
        y2: clientY - rect.top,
        isDrawing: true,
    };
};

const drawSelection = event => {
    event.preventDefault();

    if (!imageSrc.value || !canvasSelection.value.isDrawing) return;

    const clientX = event.touches ? event.touches[0].clientX : event.clientX;
    const clientY = event.touches ? event.touches[0].clientY : event.clientY;
    const rect = canvasOutput.value.getBoundingClientRect();
    canvasSelection.value.x2 = clientX - rect.left;
    canvasSelection.value.y2 = clientY - rect.top;

    // Löschen Sie den vorherigen Zustand des Canvas
    const context = canvasOutput.value.getContext('2d');
    context.clearRect(0, 0, canvasOutput.value.width, canvasOutput.value.height);

    // Stellen Sie das vorherige Bild wieder her, bevor das neue Rechteck gezeichnet wird
    if (canvasImage.value) {
        context.drawImage(canvasImage.value, 0, 0);
    }

    // Zeichnen Sie das neue Rechteck
    context.beginPath();
    context.strokeStyle = 'red'; // Sie können die Farbe des Rahmens ändern
    context.rect(canvasSelection.value.x1, canvasSelection.value.y1, canvasSelection.value.x2 - canvasSelection.value.x1, canvasSelection.value.y2 - canvasSelection.value.y1);
    context.stroke();
};

const endSelection = event => {
    event.preventDefault();

    const selection = canvasSelection.value;
    if (selection && selection.isDrawing) {
        const originalCanvas = canvasOutput.value;
        const context = originalCanvas.getContext('2d');

        // Berechnen der Größe und Position des Auswahlrechtecks
        const width = selection.x2 - selection.x1;
        const height = selection.y2 - selection.y1;

        // Erstellen eines neuen Canvas-Elements
        const newCanvas = document.createElement('canvas');
        newCanvas.width = width;
        newCanvas.height = height;
        const newContext = newCanvas.getContext('2d');

        // Kopieren des ausgewählten Bereichs in das neue Canvas
        newContext.drawImage(originalCanvas, selection.x1, selection.y1, width, height, 0, 0, width, height);

        // Ersetzen des originalen Canvas durch das neue Canvas

        selection.isDrawing = false;
        recognizeTextFromCanvas(newCanvas);
    }
};
async function recognizeTextFromCanvas(canvas) {
    const canvasToText = canvas ? canvas : canvasOutput.value;

    if (canvasToText) {
        try {
            //get image from canvas for tesseract
            // enhanceContrast(canvasToText);

            const image = canvasToText.toDataURL();
            // console.log(image);
            const result = await Tesseract.recognize(image, 'eng', {
                tessedit_char_whitelist: '0123456789',
                // tessedit_pageseg_mode: 4, // PSM.SINGLE_COLUMN (4)war okay, 6
            });

            // splitte den text \n und schriewbe die zahlen in array
            const text = result.data?.text?.split('\n');
            //entferne, das letzte lement wenn es leer ist
            if (text[text.length - 1] === '') {
                text.pop();
            }
            console.info('Ergebnis:', text);
            alert(text.join(', '));
        } catch (error) {
            console.error('Fehler bei der Texterkennung:', error);
        } finally {
            canvasSelection.value.x1 = null;
            canvasSelection.value.y1 = null;
            canvasSelection.value.x2 = null;
            canvasSelection.value.y2 = null;
        }
    }
}
</script>

<style>
.image-container {
    position: relative;
    display: inline-block;
}

.image-container img {
    display: block;
    max-width: 100%;
    height: auto;
}

.overlay {
    position: absolute;
    border: 2px dashed blue;
    cursor: pointer;
}
</style>
