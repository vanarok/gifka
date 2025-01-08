<script setup lang="ts">
import axios from "axios";
import { useInfiniteQuery } from "@tanstack/vue-query";

import Masonry from "masonry-layout";
import imagesLoaded from "imagesloaded";
import { mdiFileImageOutline, mdiMagnify } from "@mdi/js";
import { ref, watch } from "vue";
import { useScroll } from "@vueuse/core";

const vMasonry = {
  mounted: (el, binding) => {
    console.log("Initializing Masonry");
    el.masonryInstance = new Masonry(el, binding.value || {});
  },
  updated: (el) => {
    // Обновление расположения при загрузке изображений
    imagesLoaded(el, () => {
      el.masonryInstance.reloadItems();
      el.masonryInstance.layout();
    });
  },
  unmounted: (el) => {
    // Уничтожение Masonry при удалении элемента
    el.masonryInstance.destroy();
  },
};

function getImageHeightWidth(
  url: string,
): Promise<{ height: number; width: number }> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => {
      resolve({ height: img.height, width: img.width });
    };
    img.onerror = () => {
      reject(new Error("Failed to load image"));
    };
    img.src = url;
  });
}

async function mapGifsHeightWidth(gifsData: { result: any[] }) {
  return Promise.all(
    gifsData.result.map(async (gif) => {
      try {
        const { height, width } = await getImageHeightWidth(gif.cloudSource300);
        return { ...gif, height, width };
      } catch {
        return false;
      }
    }),
  );
}

async function getGifs({ pageParam = 0 }) {
  const skip = pageParam;
  const { data } = await axios.get(
    `https://gifs.ru/api/v1/Popular/Random?contentType=1&skip=${skip}&take=26`,
  );
  return {
    data: (await mapGifsHeightWidth(data)).filter((gif) => gif.fileType === 1),
    nextPage: skip + 10,
  };
}

const { data, fetchNextPage, isFetching } = useInfiniteQuery({
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage?.nextPage,
  queryKey: ["gifs"],
  queryFn: getGifs,
  select: (data) => [...data.pages.flatMap((page) => page.data)],
  refetchOnWindowFocus: false,
});

const { y } = useScroll(window, {
  throttle: 500,
  behavior: "smooth",
});

watch(y, () => {
  const maxScrollY = document.documentElement.scrollHeight - window.innerHeight;
  const offset = 200;
  console.log(y.value, maxScrollY - offset, isFetching.value);
  if (y.value > maxScrollY - offset && !isFetching.value) {
    console.log("fetch");

    fetchNextPage();
  }
});

async function searchGifs({ pageParam = 0 }) {
  const skip = pageParam;
  const { data } = await axios.post(`https://gifs.ru/api/v1/Gif/GetGifs`, {
    tags: searchInput.value.split(" "),
    skip,
    take: 10,
  });
  return {
    data: (await mapGifsHeightWidth(data)).filter((gif) => gif.fileType === 1),
    nextPage: skip + 10,
  };
}

const searchInput = ref("");

const { data: searchResults } = useInfiniteQuery({
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage?.nextPage,
  enabled: () => !!searchInput.value,
  queryKey: ["search", searchInput],
  queryFn: searchGifs,
  select: (data) => [...data.pages.flatMap((page) => page.data)],
  refetchOnWindowFocus: false,
});
</script>

<template>
  <v-app>
    <v-app-bar
      color="#121212"
      flat
      scroll-behavior="hide"
      extension-height="80"
    >
      <template #extension>
        <v-container max-width="1040" width="1040">
          <v-row no-gutters>
            <v-col cols="2">
              <v-row no-gutters align="center">
                <v-app-bar-nav-icon min-width="48" width="48">
                  <v-icon
                    :icon="mdiFileImageOutline"
                    class="glitch"
                    color="primary"
                    size="xxx-large"
                  ></v-icon>
                </v-app-bar-nav-icon>
                <v-app-bar-title
                  class="text-h4 font-weight-black align-content-center"
                >
                  GIFKA
                </v-app-bar-title>
              </v-row>
            </v-col>
            <v-col cols="10">
              <v-text-field
                min-width="840"
                width="840"
                v-model="searchInput"
                placeholder="Search all the gifs"
                clearable
                hide-details
                variant="solo"
                :append-inner-icon="mdiMagnify"
              />
            </v-col>
          </v-row>
        </v-container>
      </template>
    </v-app-bar>

    <v-main>
      <v-container min-height="105vh" max-width="1040" width="1040">
        <div
          class="grid"
          v-masonry="{
            itemSelector: '.grid-item',
            transitionDuration: 0,
            gutter: 10,
            percentPosition: true,
          }"
        >
          <div v-if="searchInput">
            <div
              class="grid-item"
              v-for="(gif, index) in searchResults"
              :key="index"
            >
              <img
                :alt="gif.tags"
                :src="gif.cloudSource300"
                :width="gif.width"
                :height="gif.height"
              />
            </div>
          </div>
          <div v-else>
            <div class="grid-item" v-for="(gif, index) in data" :key="index">
              <img
                :alt="gif.tags"
                :src="gif.cloudSource300"
                :width="gif.width"
                :height="gif.height"
              />
            </div>
          </div>
        </div>
      </v-container>
      <v-row justify="center">
        <v-col cols="auto">
          <div class="loading-indicator">
            <div></div>
            <div></div>
            <div></div>
            <div></div>
          </div>
        </v-col>
      </v-row>
    </v-main>
  </v-app>
</template>

<style scoped>
.grid-item {
  width: 244.5px; /* Ugly hack for masonry-layout */
  margin-bottom: 10px;
}

.grid-item img {
  display: block;
  width: 100%; /* Изображение адаптируется к ширине блока */
  height: auto; /* Сохраняются пропорции */
  border-radius: 4px; /* Закругленные углы (необязательно) */
}

/*glitch text logo animation */
@keyframes glitch {
  0% {
    transform: translate(0);
  }
  20% {
    transform: translate(-3px, 3px);
  }
  40% {
    transform: translate(-3px, -3px);
  }
  60% {
    transform: translate(3px, 3px);
  }
  80% {
    transform: translate(3px, -3px);
  }
  100% {
    transform: translate(0);
  }
}

.glitch {
  animation: glitch 1s infinite;
}

.loading-indicator {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 50px;
}

.loading-indicator div {
  width: 0;
  height: 0;
  border-left: 15px solid transparent;
  border-right: 15px solid transparent;
  border-bottom: 25px solid;
  margin: 0 10px;
  animation:
    loading 1s ease-in-out infinite,
    scaling 1s ease-in-out infinite;
}

.loading-indicator div:nth-child(1) {
  animation-delay: 0s, 0s;
  border-bottom-color: mediumslateblue;
  rotate: 45deg;
}

.loading-indicator div:nth-child(2) {
  animation-delay: 0.2s, 0.2s;
  border-bottom-color: mediumorchid;
  rotate: -45deg;
}

.loading-indicator div:nth-child(3) {
  animation-delay: 0.4s, 0.4s;
  border-bottom-color: mediumseagreen;
  rotate: 10deg;
}

.loading-indicator div:nth-child(4) {
  animation-delay: 0.6s, 0.6s;
  border-bottom-color: orangered;
  rotate: -90deg;
}

@keyframes loading {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

@keyframes scaling {
  0%,
  100% {
    transform: scale(0.8);
  }
  50% {
    transform: scale(1.5);
  }
}
</style>
