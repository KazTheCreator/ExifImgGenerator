<script setup lang="ts">
import { ref, computed, watch } from "vue";
import JSZip from "jszip";
import piexif from "piexifjs";
import { faker } from "@faker-js/faker";

/* ─── constants ──────────────────────────────────────────────── */
const MAX_RAM_BYTES = 500 * 1024 * 1024; // 500 MB

/* ─── reactive state ──────────────────────────────────────────────── */
const count = ref(1);
const format = ref<"jpg" | "png">("jpg");
const formatItems = [
  { label: "JPEG", value: "jpg" },
  { label: "PNG", value: "png" },
];
const randomSize = ref(true);
const width = ref(1920);
const height = ref(1080);
const withExif = ref(false);
const bgColor = ref<string>("");

const targetSize = ref(0); // 0 = auto, else bytes
const targetSizeItems = [
  { label: "Auto", value: 0 },
  { label: "1 MB", value: 1 * 1024 * 1024 },
  { label: "5 MB", value: 5 * 1024 * 1024 },
  { label: "10 MB", value: 10 * 1024 * 1024 },
];

const listView = ref(true);

const isGenerating = ref(false);
const cancelRequested = ref(false);
const stepIdx = ref(0);

const macePublicImage = "/mace-cleaned-cropped.png";

const statusSteps = [
  "Initializing canvas…",
  "Generating dimensions…",
  "Applying background color…",
  "Drawing placeholder text…",
  "Injecting EXIF metadata…",
  "Generating GeoLocation data…",
  "Rendering image textures…",
  "Finalizing image details…",
  "Compressing image data…",
  "Preparing ZIP archive…",
  "Export ready! Download your images 💾",
];

const landscapeSizes = [
  [1920, 1080],
  [1280, 720],
  [3840, 2160],
  [2560, 1440],
];
const portraitSizes = [
  [1080, 1920],
  [720, 1280],
  [2160, 3840],
  [1440, 2560],
];

const setSize = (size: [number, number]) => {
  width.value = size[0];
  height.value = size[1];
};

type GenImg = {
  name: string;
  blob: Blob;
  url: string;
  exif: any | null;
  uuid: string;
  width: number;
  height: number;
};
const images = ref<GenImg[]>([]);

/* ─── RAM safeguard logic ────────────────────────────────────────── */
const estimatedImageSize = computed(() => targetSize.value || 300 * 1024); // fallback: 300KB per image if auto
const maxCountAllowed = computed(() =>
  Math.max(1, Math.floor(MAX_RAM_BYTES / estimatedImageSize.value))
);
const totalEstimatedBytes = computed(
  () => count.value * estimatedImageSize.value
);
const isOverRamLimit = computed(
  () => totalEstimatedBytes.value > MAX_RAM_BYTES
);

/* ─── helpers ─────────────────────────────────────────────────────── */
const randInt = (min: number, max: number) =>
  Math.floor(Math.random() * (max - min + 1)) + min;

const pickSize = () => {
  const randomOffset = () => Math.floor(Math.random() * 101) - 50;

  if (randomSize.value) {
    const [baseWidth, baseHeight] =
      Math.random() > 0.5
        ? landscapeSizes[Math.floor(Math.random() * landscapeSizes.length)]
        : portraitSizes[Math.floor(Math.random() * portraitSizes.length)];

    return [baseWidth + randomOffset(), baseHeight + randomOffset()];
  } else {
    return [width.value, height.value];
  }
};

const saveBlob = (blob: Blob, filename: string) => {
  const link = Object.assign(document.createElement("a"), {
    href: URL.createObjectURL(blob),
    download: filename,
  });
  link.click();
  URL.revokeObjectURL(link.href);
};

const deleteImage = (idx: number) => images.value.splice(idx, 1);

// Inflate blob to target size by appending dummy data
function inflateBlob(blob: Blob, targetBytes: number): Promise<Blob> {
  if (!targetBytes || blob.size >= targetBytes) return Promise.resolve(blob);
  const padSize = targetBytes - blob.size;
  const pad = new Uint8Array(padSize).fill(0);
  return Promise.resolve(new Blob([blob, pad], { type: blob.type }));
}

/* ─── generation logic ────────────────────────────────────────────── */
async function generateImages() {
  if (isOverRamLimit.value) return;
  stepIdx.value = 0;
  isGenerating.value = true;
  cancelRequested.value = false;
  images.value = [];

  for (let i = 0; i < count.value; i++) {
    if (cancelRequested.value) break;
    stepIdx.value = Math.min(Math.floor(((i + 1) / count.value) * 10), 9);

    const [w, h] = pickSize();
    const uuid = crypto.randomUUID().split("-")[0];

    /* draw canvas */
    const canvas = Object.assign(document.createElement("canvas"), {
      width: w,
      height: h,
    });
    const ctx = canvas.getContext("2d")!;
    ctx.fillStyle = bgColor.value || `hsl(${randInt(0, 359)}, 40%, 85%)`;
    ctx.fillRect(0, 0, w, h);

    ctx.fillStyle = "#000";
    ctx.textAlign = "center";
    ctx.textBaseline = "middle";
    ctx.font = `${Math.round(w / 20)}px sans-serif`;
    ctx.fillText(`${w} × ${h}`, w / 2, h / 2);

    ctx.textAlign = "left";
    ctx.textBaseline = "top";
    ctx.font = `${Math.round(w / 45)}px monospace`;
    ctx.globalAlpha = 0.9;
    ctx.fillText(uuid, 12, 8);
    ctx.globalAlpha = 1;

    const mime = format.value === "jpg" ? "image/jpeg" : "image/png";
    let dataURL = canvas.toDataURL(mime, 0.9);
    let exifObj: any | null = null;

    if (withExif.value && format.value === "jpg") {
      exifObj = {
        "0th": {
          [piexif.ImageIFD.Make]: faker.company.name(),
          [piexif.ImageIFD.Model]: faker.commerce.productName(),
          [piexif.ImageIFD.Software]: "Placeholder Forge",
        },
        Exif: {
          [piexif.ExifIFD.DateTimeOriginal]: faker.date
            .recent()
            .toISOString()
            .slice(0, 19)
            .replace("T", " "),
        },
        GPS: {
          [piexif.GPSIFD.GPSLatitudeRef]: "N",
          [piexif.GPSIFD.GPSLatitude]: piexif.GPSHelper.degToDmsRational(
            randInt(0, 90) + Math.random()
          ),
          [piexif.GPSIFD.GPSLongitudeRef]: "E",
          [piexif.GPSIFD.GPSLongitude]: piexif.GPSHelper.degToDmsRational(
            randInt(0, 180) + Math.random()
          ),
        },
      };
      dataURL = piexif.insert(piexif.dump(exifObj), dataURL);
    }

    let blob = await (await fetch(dataURL)).blob();
    blob = await inflateBlob(blob, targetSize.value);

    images.value.push({
      name: `image_${i + 1}_${uuid}_${w}x${h}.${format.value}`,
      blob,
      url: URL.createObjectURL(blob),
      exif: exifObj,
      uuid,
      width: w,
      height: h,
    });
  }

  if (!cancelRequested.value) stepIdx.value = 10;
  isGenerating.value = false;
}

/* ─── Watchers to enforce max count ──────────────────────────────── */
watch([targetSize, count], () => {
  if (count.value > maxCountAllowed.value) {
    count.value = maxCountAllowed.value;
  }
});

/* ─── Download & misc ────────────────────────────────────────────── */
const downloadZip = async () => {
  const zip = new JSZip();
  images.value.forEach((i) => zip.file(i.name, i.blob));
  saveBlob(await zip.generateAsync({ type: "blob" }), "images.zip");
};

const resetColor = () => (bgColor.value = null);
const cancelGeneration = () => (cancelRequested.value = true);
</script>

<template>
  <UContainer class="">
    <!-- ───── Hero ───────────────────────────────────────────── -->
    <div class="text-center space-y-3 mb-4 mt-4">
      <h1 class="text-4xl font-extrabold tracking-tight mb-1">
        Placeholder Forge
      </h1>
      <!-- <NuxtImg
        :src="macePublicImage"
        class=""
        alt="Placeholder Forge logo"
      /> -->
      <p class="text-gray-500 max-w-xl mx-auto">
        Craft colourful placeholder images, sprinkle random EXIF magic, and
        download them one-by-one or zipped together.
      </p>
    </div>

    <!-- ───── Main row ──────────────────────────────────────── -->
    <div class="grid lg:grid-cols-2 gap-8">
      <!-- Settings -->
      <UCard v-auto-animate class="self-start h-fit">
        <template #header>Generator settings</template>

        <UFormField label="Number of images">
          <div class="space-y-2">
            <UInput
              v-model.number="count"
              type="number"
              :min="1"
              :max="maxCountAllowed"
              size="lg"
              class="w-full"
              :disabled="isGenerating"
            />
            <div class="text-xs text-gray-500">
              Max allowed: {{ maxCountAllowed }} (based on target file size)
            </div>
            <div
              v-if="isOverRamLimit"
              class="text-xs text-red-600 font-semibold"
            >
              ⚠️ Total estimated RAM usage ({{
                (totalEstimatedBytes / (1024 * 1024)).toFixed(1)
              }}
              MB) exceeds 500 MB limit!
            </div>
            <div v-else class="text-xs text-gray-500">
              Estimated RAM usage:
              {{ (totalEstimatedBytes / (1024 * 1024)).toFixed(1) }} MB
            </div>
            <!-- quick presets -->
            <div class="flex gap-2 flex-wrap">
              <UBadge
                v-for="preset in [5, 25, 50, 100]"
                :key="preset"
                class="cursor-pointer"
                clickable
                variant="outline"
                @click="count = Math.min(preset, maxCountAllowed)"
              >
                {{ preset }}
              </UBadge>
            </div>
          </div>
        </UFormField>

        <UFormField label="Random size (≤ 4 K px)" class="mt-4">
          <USwitch v-model="randomSize" :disabled="isGenerating" />
        </UFormField>

        <div v-if="!randomSize" class="grid grid-cols-2 gap-4 mt-4">
          <UFormField label="Width (px)">
            <UInput
              v-model.number="width"
              class="w-full"
              type="number"
              :min="1"
              :max="3840"
              :disabled="isGenerating"
            />
          </UFormField>
          <UFormField label="Height (px)">
            <UInput
              v-model.number="height"
              class="w-full"
              type="number"
              :min="1"
              :max="3840"
              :disabled="isGenerating"
            />
          </UFormField>
        </div>
        <div v-if="!randomSize" class="flex gap-2 flex-wrap mt-2">
          <!-- Landscape Sizes -->
          <UBadge
            v-for="(size, idx) in landscapeSizes"
            :key="'landscape-' + idx"
            class="cursor-pointer"
            clickable
            variant="outline"
            @click="setSize(size)"
          >
            <UIcon
              name="i-heroicons-device-phone-mobile"
              class="mr-1 rotate-90"
            />
            {{ size[0] }} × {{ size[1] }}
          </UBadge>

          <!-- Portrait Sizes -->
          <UBadge
            v-for="(size, idx) in portraitSizes"
            :key="'portrait-' + idx"
            class="cursor-pointer"
            clickable
            variant="outline"
            @click="setSize(size)"
          >
            <UIcon name="i-heroicons-device-phone-mobile" class="mr-1" />
            {{ size[0] }} × {{ size[1] }}
          </UBadge>
        </div>

        <UFormField label="Image format" class="mt-4">
          <URadioGroup
            v-model="format"
            :items="formatItems"
            :disabled="isGenerating"
          />
        </UFormField>

        <UFormField
          v-if="format === 'jpg'"
          label="Include randomised EXIF"
          class="mt-4"
        >
          <USwitch v-model="withExif" :disabled="isGenerating" />
        </UFormField>

        <UFormField label="Background colour" class="mt-4">
          <template #default>
            <div class="flex items-center gap-3">
              <UPopover>
                <UButton
                  :label="bgColor || 'Choose color'"
                  color="neutral"
                  variant="outline"
                >
                  <template #leading>
                    <span
                      :style="{ backgroundColor: bgColor || 'transparent' }"
                      class="size-3 rounded-full border"
                    />
                  </template>
                </UButton>

                <template #content>
                  <UColorPicker
                    v-model="bgColor"
                    class="p-2"
                    clearable
                    :throttle="200"
                    :disabled="isGenerating"
                  />
                </template>
              </UPopover>
              <UButton
                size="xs"
                variant="outline"
                :disabled="isGenerating || !bgColor"
                @click="resetColor"
              >
                Reset
              </UButton>
            </div>
          </template>
        </UFormField>

        <UFormField label="Target file size" class="mt-4">
          <URadioGroup
            v-model="targetSize"
            :items="targetSizeItems"
            :disabled="isGenerating"
          />
        </UFormField>

        <!-- actions -->
        <div class="flex gap-4 mt-6">
          <UButton
            color="primary"
            :disabled="isGenerating || isOverRamLimit"
            @click="generateImages"
            >Generate</UButton
          >
          <UButton
            v-if="isGenerating"
            color="warning"
            variant="outline"
            @click="cancelGeneration"
            >Cancel</UButton
          >
          <UButton
            variant="outline"
            :disabled="isGenerating || !images.length"
            @click="downloadZip"
            >Download ZIP</UButton
          >
        </div>

        <UProgress
          v-if="isGenerating"
          v-model="stepIdx"
          :max="statusSteps"
          size="sm"
          class="mt-4"
        />
      </UCard>

      <!-- Gallery -->
      <UCard>
        <template #header>
          <div class="flex items-center justify-between">
            <span>Preview gallery</span>
            <div class="flex items-center gap-2 text-sm">
              Tile
              <USwitch v-model="listView" />
              <!-- ON = list -->
              List
            </div>
          </div>
        </template>

        <div class="mb-4">
          <p
            v-if="!images.length && !isGenerating"
            class="text-gray-500 text-center"
          >
            No images yet…
          </p>
          <p
            v-else-if="isGenerating"
            class="text-gray-500 text-center animate-pulse"
          >
            {{ statusSteps[stepIdx] }}
          </p>
        </div>

        <!-- Tile view -->
        <div
          v-if="!listView && images.length"
          v-auto-animate
          class="masonry-grid"
        >
          <div
            v-for="(img, idx) in images"
            :key="img.name"
            class="masonry-item relative rounded overflow-hidden shadow bg-list"
          >
            <div class="absolute right-2 top-2 z-10 flex gap-1">
              <UButton
                size="xs"
                color="primary"
                variant="solid"
                @click="saveBlob(img.blob, img.name)"
              >
                <UIcon name="i-heroicons-arrow-down-tray" />
              </UButton>
              <UButton
                size="xs"
                color="error"
                variant="solid"
                @click="deleteImage(idx)"
              >
                <UIcon name="i-heroicons-trash" />
              </UButton>
            </div>
            <NuxtImg
              :src="img.url"
              :alt="img.name"
              class="w-full max-h-60 object-cover bg-gray-50"
              loading="lazy"
            />
          </div>
        </div>

        <!-- List view -->
        <div
          v-else-if="listView && images.length"
          v-auto-animate
          class="space-y-3 overflow-hidden"
        >
          <div
            v-for="(img, idx) in images"
            :key="img.name"
            class="grid items-center gap-3 rounded bg-list p-2"
            style="grid-template-columns: auto 1fr auto auto auto"
          >
            <!-- thumbnail -->
            <NuxtImg
              :src="img.url"
              class="w-12 h-12 object-cover bg-gray-50 rounded flex-shrink-0"
              :alt="img.name"
              loading="lazy"
            />

            <!-- text (only column that can grow) -->
            <div class="min-w-0">
              <p class="text-sm truncate">{{ img.name }}</p>
              <p class="text-xs text-gray-500 flex items-center gap-1">
                <UIcon
                  name="i-heroicons-device-phone-mobile"
                  :class="img.width >= img.height ? 'rotate-90' : ''"
                />
                {{ img.width }} × {{ img.height }} px
              </p>
            </div>

            <UBadge
              v-if="img.exif"
              color="primary"
              variant="subtle"
              class="flex-shrink-0"
            >
              EXIF
            </UBadge>

            <UButton
              size="xs"
              color="primary"
              variant="solid"
              class="flex-shrink-0"
              @click="saveBlob(img.blob, img.name)"
            >
              <UIcon name="i-heroicons-arrow-down-tray" />
            </UButton>

            <UButton
              size="xs"
              color="error"
              variant="solid"
              class="flex-shrink-0"
              @click="deleteImage(idx)"
            >
              <UIcon name="i-heroicons-trash" />
            </UButton>
          </div>
        </div>
      </UCard>
    </div>

    <!-- Disclaimer -->
    <UAccordion
      :items="[{ label: 'ℹ️  About & Disclaimer', description: 'Details' }]"
      class="mt-8 mb-8"
    >
      <template #content>
        <p class="text-sm leading-relaxed">
          Placeholder Forge runs entirely in your browser. Images are painted
          with the HTML5 canvas; optional EXIF metadata is injected using
          <code>piexifjs</code>. No files or data ever leave your device.<br /><br />
          <strong>JPEG only:</strong> EXIF is ignored for PNG outputs.
        </p>
      </template>
    </UAccordion>

    <!-- LICENSE / COPYRIGHT -->
    <p class="mt-4 mb-8 text-xs text-gray-500">
      © 2025 Alexander Müller / KazTheCreator
      <br />
      MIT LICENSE
    </p>
  </UContainer>
</template>

<style scoped>
img {
  transition: transform 0.2s ease;
}
img:hover {
  transform: scale(1.05);
}
pre {
  font-family: ui-monospace, SFMono-Regular, Consolas, Menlo, monospace;
}

.cursor-pointer {
  cursor: pointer;
}

.bg-list {
  background: #00000010;
}

.masonry-grid {
  column-count: 2;
  column-gap: 1rem;
  overflow: hidden;
}
@media (min-width: 768px) {
  .masonry-grid {
    column-count: 3;
  }
}
.masonry-item {
  break-inside: avoid;
  margin-bottom: 1rem;
}
</style>
