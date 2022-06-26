<template>
  <div class="w-screen h-screen">
    <canvas ref="canvas" width="800" height="800"></canvas>
  </div>

  <div class="slidecontainer">
    <input
      type="range"
      min="1"
      max="100"
      value="0"
      class="slider"
      id="myRange"
      @input="onSliderChange"
    />
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from "vue";
const canvas = ref<HTMLCanvasElement | null>(null);

interface Coords {
  row: number;
  col: number;
}

function getCoords(i: number): Coords {
  const row = Math.floor(i / 800);
  const col = i % 800;

  return { row, col };
}

function createColorMap({ data }: ImageData) {
  let counter = 0;
  const map: Record<string, Coords[]> = {};
  for (let i = 0; i < data.length; i += 4) {
    const r = data[i];
    const g = data[i + 1];
    const b = data[i + 2];
    const a = data[i + 3];

    const coords = getCoords(counter);
    const key = `${r}-${g}-${b}-${a}`;
    map[key] ||= [];
    map[key].push(coords);

    counter++;
  }

  return map;
}

interface Bounding {
  x: number;
  y: number;
  width: number;
  height: number;
}

function getBoundsForCoords(coords: Coords[]): Bounding {
  let topRow = Infinity - 1;
  let bottomRow = 0;
  let leftCol = Infinity - 1;
  let rightCol = 0;

  coords.forEach(({ row, col }) => {
    if (row < topRow) {
      topRow = row;
    }

    if (row > bottomRow) {
      bottomRow = row;
    }

    if (col < leftCol) {
      leftCol = col;
    }

    if (col > rightCol) {
      rightCol = col;
    }
  });

  return {
    x: leftCol,
    y: topRow,
    width: rightCol - leftCol,
    height: bottomRow - topRow,
  };
}

function drawAround(
  { x, y, width, height }: Bounding,
  key: string,
  ctx: CanvasRenderingContext2D,
  imageData: ImageData
) {
  const fgColor = `rgba(${key.split("-").join(",")})`;

  // @ts-ignore
  window.setWidth = function (n: number) {
    ctx.putImageData(imageData, 0, 0);
    ctx.lineWidth = n;
    ctx.strokeStyle = fgColor;
    ctx.strokeRect(x - n / 2, y - n / 2, width + n, height + n);
  };
}

function onSliderChange(e: any) {
  const val = e.target.value;

  // @ts-ignore
  window.setWidth(parseInt(val));
}

function onLoadImage(img: string) {
  const ctx = canvas.value?.getContext("2d");

  const image = new Image();
  image.src = img;
  image.onload = function () {
    if (ctx) {
      ctx.fillStyle = "#f06";
      ctx.fillRect(0, 0, 800, 800);

      const width = image.naturalWidth / 2;
      const height = image.naturalHeight / 2;

      ctx.drawImage(image, 400 - width / 2, 400 - height / 2, width, height);

      const imageData = ctx.getImageData(0, 0, 800, 800);

      const map = createColorMap(imageData);

      const keys = Object.keys(map).sort((a, b) =>
        map[a].length > map[b].length ? 1 : 0
      );

      const coords = map[keys[1]];
      const bounds = getBoundsForCoords(coords);

      drawAround(bounds, keys[1], ctx, imageData);
    }
  };
}

onMounted(() => {
  if (!canvas.value) return;
  const ctx = canvas.value?.getContext("2d");
  if (ctx) {
    ctx.fillStyle = "#f06";
    ctx.fillRect(0, 0, 800, 800);
  }

  //   onLoadImage(img);

  document.onpaste = function (pasteEvent) {
    // consider the first item (can be easily extended for multiple items)
    var item = pasteEvent.clipboardData?.items[0];
    if (!item) return;
    if (item.type.indexOf("image") === 0) {
      var blob = item.getAsFile();

      var reader = new FileReader();
      reader.onload = function (event) {
        if (event.target?.result) {
          onLoadImage(event.target?.result as string);
        }
      };

      if (blob) {
        reader.readAsDataURL(blob);
      }
    }
  };
});
</script>
