<template>
  <main id="page-feed" class="flex flex-col pt-12 sm:pt-14">
    <!-- Sub Navigation & Banner -->
    <section class="flex-col" :class="route.params.board ? 'flex' : 'hidden md:flex'">
      <NavigationNavbarSub :links="links" class="sm:order-first" />

      <div class="order-first sm:order-last container mx-auto max-w-8xl grid grid-cols-12 sm:mt-16 sm:px-4 md:px-6">
        <!-- Banner -->
        <LazyCardsBoardBanner v-if="route.params.board" :board="dummyBoard" class="col-span-full" />
        <LazyCardsBanner v-else title="Feed" sub-title="Welcome to the awesome and exciting front page."
          image-url="/img/artwork/front-page.jpeg" class="col-span-full" />
      </div>


    </section>
    <!-- Main Content -->
    <section class="container mx-auto max-w-8xl grid grid-cols-12 sm:my-6 sm:px-4 md:px-6">
      <div class="col-span-full flex gap-6">
        <div class="w-full">
          <!-- Submission Form -->
          <!-- Sorts & View Options -->
          <div
            class="flex items-center mb-4 p-2.5 sm:px-4 sm:py-3 bg-gray-100 dark:bg-gray-950 border-b sm:border sm:shadow-inner-white sm:rounded-md dark:border-gray-800 dark:shadow-none">
            <MenusSort />
            <div class="ml-auto flex space-x-2">
              <button class="ml-auto" @click="preferCardView = true">
                <!-- Rows Icon -->
                <svg xmlns="http://www.w3.org/2000/svg" class="w-5 h-5"
                  :class="preferCardView ? 'text-primary' : 'text-gray-500'" viewBox="0 0 24 24" stroke-width="2"
                  stroke="currentColor" fill="none" stroke-linecap="round" stroke-linejoin="round">
                  <path stroke="none" d="M0 0h24v24H0z" fill="none"></path>
                  <rect x="4" y="4" width="16" height="6" rx="2"></rect>
                  <rect x="4" y="14" width="16" height="6" rx="2"></rect>
                </svg>
              </button>
              <button class="ml-auto" @click="preferCardView = false">
                <!-- Cards Icon -->
                <svg xmlns="http://www.w3.org/2000/svg" class="w-5 h-5"
                  :class="preferCardView ? 'text-gray-500' : 'text-primary'" viewBox="0 0 24 24" stroke-width="2"
                  stroke="currentColor" fill="none" stroke-linecap="round" stroke-linejoin="round">
                  <path stroke="none" d="M0 0h24v24H0z" fill="none"></path>
                  <rect x="4" y="4" width="16" height="16" rx="2"></rect>
                  <line x1="4" y1="12" x2="20" y2="12"></line>
                </svg>
              </button>
            </div>
          </div>
          <!-- Posts -->
          <LazyListsPosts v-if="posts?.length" :posts="posts" :isCompact="!preferCardView" :isLoading="pending"
            :hasError="error" />
          <!-- Empty State -->
          <div v-else
            class="px-4 py-24 text-center text-gray-500 bg-white dark:bg-gray-950 border-y sm:border sm:rounded-md sm:shadow-inner-xs dark:border-gray-800">
            <p>
              <span class="font-medium"> There are no posts </span>
              <br />
              but you can be the first...
            </p>
          </div>
          <!-- Pagination -->
          <div v-if="totalPages > 1" class="w-full mt-4 px-2.5 sm:px-0">
            <LazyNavigationPaginate :total-pages="totalPages" :per-page="limit" :current-page="page" />
          </div>
        </div>
        <!-- Sidebar -->
        <component :is="route.params.board ? SidebarBoard : Sidebar" :board="route.params.board ? dummyBoard : null" />
      </div>
    </section>
  </main>
</template>

<script setup>
import { usePostsStore } from "@/stores/StorePosts";
import { getListing } from "@/composables/listing";
import { useLoggedInUser } from "@/stores/StoreAuth";

console.log("hi from the feed page");

// Import sidebar components
const Sidebar = defineAsyncComponent(() =>
  import("@/components/containers/Sidebar")
);
const SidebarBoard = defineAsyncComponent(() =>
  import("@/components/containers/SidebarBoard")
);

console.log("and hi again after some imports");

const router = useRouter();
const route = useRoute();
const userStore = useLoggedInUser();
const v = userStore.user;

console.log("stores have been set up");

definePageMeta({
  alias: ["", "/+:board?/feed/:sort?", "/+:board?/:sort?"],
  key: (route) => route.fullPath,
});

console.log("page meta is good");

const preferCardView = useCookie("preferCardView") ?? false;

console.log("cookies are alright");

// Pagination
const page = computed(() => Number.parseInt(route.query.page) || 1);
const limit = computed(() => Number.parseInt(route.query.limit) || 25);

console.log("no problems with the query params either");

const dummyBoard = {
  name: route.params.board,
  title: "Board title, defaults to board name",
  description: "Board short description, max 256 chars",
  followers_count: 304,
  posts_count: 1902,
  comments_count: 13929,
  created_at: "2022-11-27T19:31:27.730335",
  icon_url: "https://i.ibb.co/r4D7WWs/scott-goodwill-y8-Ngwq34-Ak-unsplash.jpg",
  banner_url: "/img/artwork/front-page.jpeg",
};

console.log("dummy board has been set up");


// Posts
const postsStore = usePostsStore();
console.log("using posts store");

const sorts = [
  "hot",
  "new",
  "topall",
  "topmonth",
  "topweek",
  "topday",
  "mostcomments",
  "newcomments",
];
const sort = computed(() => {
  return sorts.includes(route.params.sort) ? route.params.sort : "hot";
});

console.log("computed sort");

const { items, totalCount, paginate, pending, error, refresh } =
  await getListing(
    {
      sort: sort.value,
      limit: limit.value,
      page: page.value,
    },
    "posts"
  );

console.log("listing awaited");

if (error.value && error.value.response) {
  throw createError({
    statusCode: 404,
    statusMessage:
      "We could not find the page you were looking for. Try better next time.",
    fatal: true,
  });
}

console.log("no error with listing request");

postsStore.posts = items;
const posts = postsStore.posts;

const totalPages = computed(() => {
  return Math.ceil(totalCount.value / limit.value || 1);
});

console.log("posts saved");

// Links for sub navbar
const links = [
  { name: "New Thread", href: "/submit" },
  { name: "House Rules", href: "/help/rules", target: "_blank" },
];

console.log("links are okay");
</script>
