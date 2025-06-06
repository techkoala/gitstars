<template>
  <div class="relative flex-auto bg-white" style="width: calc(100% - 24rem)">
    <header
      v-if="selectedRepository"
      class="flex h-12 items-center border-b border-solid border-apple-gray-200 px-6 bg-apple-gray-50"
    >
      <a
        :href="toRepositoryHref(selectedRepository)"
        class="inline-flex items-center text-[var(--text-primary)] hover:text-[var(--primary)] transition-colors duration-300"
        rel="noopener noreferrer"
      >
        <h2 class="font-sf-pro-display font-semibold">
          <svg-icon name="github" class="mr-1 text-xl" />
          <span class="font-medium">{{ selectedRepository?.owner.login }}</span> / 
          <span class="font-bold">{{ selectedRepository?.name }}</span>
          <svg-icon name="share" class="ml-1 text-sm text-[var(--primary)]" />
        </h2>
      </a>
    </header>

    <div
      v-show="!readme"
      class="flex flex-col items-center justify-center h-full"
    >
      <div class="text-center">
        <p class="mb-4 text-3xl font-sf-pro-display font-bold text-apple-gray-300">README.md</p>
        <p
          v-if="!selectedRepository"
          class="flex items-center justify-center text-sm text-[var(--text-secondary)]"
        >
          <svg-icon name="hand-left" class="mr-2 text-base" />{{
            $t('readmeTip')
          }}
        </p>
        <div v-show="loading" class="mt-6">
          <svg-icon name="loading" class="animate-spin text-2xl text-[var(--primary)]" />
        </div>
      </div>
    </div>

    <article
      v-show="readme"
      v-html="readme"
      ref="refReadme"
      class="markdown-body overflow-y-auto p-8 text-sm bg-white"
      style="height: calc(100% - 3rem)"
    />
  </div>
</template>

<script setup>
import { watchEffect, ref, nextTick } from 'vue';
import { storeToRefs } from 'pinia';
import { useRepositoryStore } from '@/store/repository';
import { getRepositoryReadme, getReadmeByMarkdown } from '@/server/github';
import { toRepostoryReadmeHref } from './tool';

const { selectedRepository } = storeToRefs(useRepositoryStore());
const readme = ref('');
const refReadme = ref(null);
const loading = ref(false);

watchEffect(async () => {
  if (!selectedRepository.value) return;
  readme.value = '';
  loading.value = true;

  // 暂存 repository id
  const { id } = selectedRepository.value;

  const { content, html_url } = await getRepositoryReadme({
    owner: selectedRepository.value.owner.login,
    name: selectedRepository.value.name,
  });
  const result = await getReadmeByMarkdown(
    decodeURIComponent(escape(atob(content))),
  );

  /**
   * 接口异步响应需要时间
   * 此时 selectedRepository 可能已变更
   */
  if (id !== selectedRepository.value.id) return;
  loading.value = false;

  /**
   * https://github.com/coder/code-server/blob/main/docs/README.md
   * =>
   * https://github.com/coder/code-server/blob/main/docs/
   */
  const urlPrefix = html_url.slice(0, -9);

  /**
   * 1. <img src="https://github.com/path/to/a.png" />
   * 2. <img src="./path/to/a.png" />
   * 3. <img src="path/to/a.png" />
   * 4. <img src=".filename/path/to/a.png" />
   * 5. <img src="/path/to/a.png" />
   */

  /**
   * 1. <a href="https://github.com/path/to/a.png" />
   * 2. <a href="./path/to/a.png" />
   * 3. <a href="path/to/a.png" />
   * 4. <a href=".filename/path/to/a.png" />
   * 5. <a href="/path/to/a.png" />
   * 6. <a href="#id-to-content" />
   */

  readme.value = result
    .replace(/<[^>]+href="([^"]+)(?=")/g, (match, p1) => {
      const a = match.slice(0, match.lastIndexOf('"') + 1);
      const b = toRepostoryReadmeHref(p1, { urlPrefix });
      return a + b;
    })
    .replace(/<[^>]+src="([^"]+)(?=")/g, (match, p1) => {
      const a = match.slice(0, match.lastIndexOf('"') + 1);
      const b = toRepostoryReadmeHref(p1, { urlPrefix });
      return (a + b).replace('/blob/', '/raw/');
    });

  nextTick(() => {
    refReadme.value.scrollTo({ top: 0 });
  });
});

const toRepositoryHref = (repository) =>
  `https://github.com/${repository.owner.login}/${repository.name}`;
</script>

<style src="github-markdown-css/github-markdown-light.css"></style>

<style>
.markdown-body {
  font-size: 14px;
  font-family: var(--sf-pro, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif);
  color: var(--text-primary);
  line-height: 1.6;
}

.markdown-body img {
  display: inline;
  border-radius: 6px;
}

.markdown-body pre {
  background-color: var(--background-light) !important;
  border-radius: 8px;
}

.markdown-body code {
  font-family: var(--sf-mono, 'SF Mono', SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', monospace);
}

.markdown-body h1, .markdown-body h2, .markdown-body h3, 
.markdown-body h4, .markdown-body h5, .markdown-body h6 {
  font-family: var(--sf-pro-display, -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif);
  font-weight: 600;
  color: var(--text-primary);
}
</style>
