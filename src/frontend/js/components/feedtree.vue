<template>
  <div class="d-flex flex-column gap-1">
    <div
      class="c-listitem d-flex user-select-none gap-2"
      role="radio"
      :aria-checked="modelValue === ''"
      @click="$emit('update:modelValue', '')">
      <div class="flex-shrink-0 d-flex">
        <v-icon name="layers" />
      </div>
      <div class="flex-grow-1 min-w-0 text-truncate">
        {{
          { unread: $t("all_unread"), starred: $t("all_starred"), "": $t("all_feeds") }[
            filterSelected
          ]
        }}
      </div>
      <div class="flex-shrink-0" v-if="filterSelected">
        <span class="ps-2 text-end opacity-50">{{ stats.total[filterSelected] }}</span>
      </div>
    </div>
    <component
      v-for="branch in branches"
      :key="branch.name"
      :is="branch.name ? 'details' : 'div'"
      :open="branch.name && branch.nodes.some(isSelected) ? true : undefined">
      <summary v-if="branch.name" class="c-listitem user-select-none">{{ branch.name }}</summary>
    <template
      v-for="node in branch.nodes"
      :key="node.type === 'folder' ? 'folder:' + node.folder.id : 'feed:' + node.feed.id">
      <template v-if="node.type === 'folder'">
        <div
          class="c-listitem d-flex user-select-none gap-2"
          role="radio"
          :aria-checked="modelValue === 'folder:' + node.folder.id"
          @click="$emit('update:modelValue', 'folder:' + node.folder.id)">
          <div class="flex-shrink-0 d-flex">
            <div @click.stop="$emit('toggle-folder', node.folder)" class="p-2 m-n2 d-inline-flex">
              <v-icon name="chevron-right" v-if="!node.folder.is_expanded" />
              <v-icon name="chevron-down" v-else-if="node.folder.is_expanded" />
            </div>
          </div>
          <div class="flex-grow-1 min-w-0 text-truncate">
            {{ node.folder.title.split(' / ').at(-1) }}
          </div>
          <div class="flex-shrink-0" v-if="filterSelected">
            <span class="ps-2 text-end opacity-50">{{
              stats.folders[node.folder.id]?.[filterSelected]
            }}</span>
          </div>
        </div>
        <div
          v-if="node.folder.is_expanded && node.feeds.length > 0"
          class="d-flex flex-column gap-1 ps-3">
          <div
            v-for="feedNode in node.feeds"
            :key="'feed:' + feedNode.feed.id"
            class="c-listitem d-flex user-select-none gap-2"
            role="radio"
            :aria-checked="modelValue === 'feed:' + feedNode.feed.id"
            @click="$emit('update:modelValue', 'feed:' + feedNode.feed.id)">
            <div class="flex-shrink-0 d-flex align-items-center">
              <v-icon class="flex-shrink-0" name="rss" v-if="!feedNode.feed.icon" />
              <span class="c-icon" v-else>
                <img :src="feedNode.feed.icon" alt="" loading="lazy" />
              </span>
            </div>
            <div class="flex-grow-1 min-w-0 text-truncate">
              {{ feedNode.feed.title }}
            </div>
            <div class="flex-shrink-0 d-flex">
              <span class="ps-2 text-end opacity-50" v-if="filterSelected">{{
                stats.feeds[feedNode.feed.id]?.[filterSelected]
              }}</span>
              <v-icon
                class="flex-shrink-0"
                :title="feedErrors[feedNode.feed.id]"
                v-if="!filterSelected && feedErrors[feedNode.feed.id]"
                name="alert-circle" />
            </div>
          </div>
        </div>
      </template>
      <template v-else-if="node.type === 'feed'">
        <div
          class="c-listitem d-flex user-select-none gap-2"
          role="radio"
          :aria-checked="modelValue === 'feed:' + node.feed.id"
          @click="$emit('update:modelValue', 'feed:' + node.feed.id)">
          <div class="flex-shrink-0 d-flex">
            <v-icon class="flex-shrink-0" name="rss" v-if="!node.feed.icon" />
            <span class="c-icon" v-else>
              <img :src="node.feed.icon" alt="" loading="lazy" />
            </span>
          </div>
          <div class="flex-grow-1 min-w-0 text-truncate">
            {{ node.feed.title }}
          </div>
          <div class="flex-shrink-0 d-flex">
            <span class="ps-2 text-end opacity-50" v-if="filterSelected">{{
              stats.feeds[node.feed.id]?.[filterSelected]
            }}</span>
            <v-icon
              class="flex-shrink-0"
              :title="feedErrors[node.feed.id]"
              v-if="!filterSelected && feedErrors[node.feed.id]"
              name="alert-circle" />
          </div>
        </div>
      </template>
    </template>
    </component>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import type { PropType } from "vue";
import type { Folder, Feed } from "../api-types";
import icon from "../components/icon.vue";

export interface TreeFeedNode {
  type: "feed";
  feed: Feed;
}
export interface TreeFolderNode {
  type: "folder";
  folder: Folder;
  feeds: TreeFeedNode[];
}
export type FeedTreeNode = TreeFolderNode | TreeFeedNode;

export default defineComponent({
  components: { "v-icon": icon },
  props: {
    tree: { type: Array as PropType<FeedTreeNode[]>, required: true },
    modelValue: { type: String, required: true },
    filterSelected: { type: String as PropType<"" | "unread" | "starred">, required: true },
    stats: {
      type: Object as PropType<{
        feeds: Record<number, { unread: number; starred: number }>;
        folders: Record<number, { unread: number; starred: number }>;
        total: { unread: number; starred: number };
      }>,
      required: true,
    },
    feedErrors: { type: Object as PropType<Record<number, string>>, required: true },
  },
  computed: {
    branches(): { name: string; nodes: FeedTreeNode[] }[] {
      const grouped = new Map<string, FeedTreeNode[]>();
      for (const node of this.tree) {
        const title = node.type === "folder" ? node.folder.title : "";
        const separator = title.indexOf(" / ");
        const name = separator < 0 ? "" : title.slice(0, separator);
        if (!grouped.has(name)) grouped.set(name, []);
        grouped.get(name)!.push(node);
      }
      return Array.from(grouped, ([name, nodes]) => ({ name, nodes }));
    },
  },
  methods: {
    isSelected(node: FeedTreeNode): boolean {
      return node.type === "feed"
        ? this.modelValue === `feed:${node.feed.id}`
        : this.modelValue === `folder:${node.folder.id}` ||
            node.feeds.some(child => this.modelValue === `feed:${child.feed.id}`);
    },
  },
  emits: ["update:modelValue", "toggle-folder"],
});
</script>
