<script setup lang="ts">
import { LibraryApi, TaskKind, TasksApi, type PlaylistRead, type PlaylistTaskStatus, type ProviderRead } from '@/api';
import AppButton from '../button/AppButton.vue';
import AppFormSpacer from '../form/AppFormSpacer.vue';
import { computed, onMounted, ref, watch } from 'vue';
import { Icon } from '@iconify/vue';
import ProviderSelector from '@/components/service/ProviderSelector.vue'
import { get_access_token, get_api_configuration } from '@/services/api';
import PlaylistSelector from './PlaylistSelector.vue';
import { useRouteQuery } from '@vueuse/router';
import { isAxiosError } from 'axios';
import AppCheckbox from '../form/AppCheckbox.vue';
import AppGroupWrapper from '../generic/AppGroupWrapper.vue';
import { decode_entity_id, encode_entity_id } from '@/utils/id.ts';

const props = defineProps<{
  providers: ProviderRead[];
}>();

const emit = defineEmits<{
  submitted: [task: PlaylistTaskStatus];
}>();

const providers = computed(() => props.providers.filter((provider) => provider.is_configured && provider.linking.linked));
const config = get_api_configuration(get_access_token());
const libraryApi = new LibraryApi(config);
const tasksApi = new TasksApi(config);
const isLoading = ref(false);
const isDryRun = ref(false);
const playlists = ref<PlaylistRead[]>([]);

const disallowSubmit = computed(() => {
  return !sourceProviderChoice.value || !targetProviderChoice.value || !sourcePlaylist.value;
});

const sourceProviderName = useRouteQuery<string>('from_provider');
const sourceProviderChoice = computed(() => {
  return providers.value.find((provider) => provider.provider_name === sourceProviderName.value);
});

const targetProviderName = useRouteQuery<string>('to_provider');
const targetProviderChoice = computed(() => {
  return providers.value.find((provider) => provider.provider_name === targetProviderName.value);
});

const fetchLibraryPlaylists = async () => {
  try {
    const response = await libraryApi.getLibraryPlaylists();

    if (response.data.items) {
      playlists.value = response.data.items;
    }
  } catch (error) {
    if (isAxiosError(error)) {
      console.error(`Failed to fetch available playlists. Error: "${error.response?.data ?? error.message}""`)
    }
  }
}

const transferablePlaylists = ref<PlaylistRead[]>([]);
const filterOnlyPlaylistsFromSourceProvider = async () => {
  const provider_name = sourceProviderChoice.value?.provider_name;

  if (!provider_name) {
    return;
  }

  transferablePlaylists.value = playlists.value.filter((p) => p.meta.provider_name === provider_name);
};

const targetablePlaylists = ref<PlaylistRead[]>([]);
const filterOnlyPlaylistsFromTargetProvider = async () => {
  const provider_name = targetProviderChoice.value?.provider_name;

  if (!provider_name) {
    return;
  }

  targetablePlaylists.value = playlists.value.filter((p) => p.meta.provider_name === provider_name);
  targetablePlaylists.value.unshift({
    title: 'Create new playlist',
    description: 'Creates a brand new playlist.',
    is_public: true,
    author: {
      primary: "tunesynctool",
    },
    meta: {
      provider_name: provider_name,
      share_url: undefined
    },
    assets: {
      cover_image: undefined,
    },
    identifiers: {
      provider_id: encode_entity_id(targetProviderChoice.value.provider_name, "client#create_new"),
    }
  })
};

const sourcePlaylistId = useRouteQuery<string | undefined>('from_playlist');
const sourcePlaylist = computed(() => {
  return transferablePlaylists.value.find((playlist) => playlist.identifiers.provider_id === sourcePlaylistId.value);
});

const targetPlaylistId = useRouteQuery<string | undefined>('to_playlist');
const targetPlaylist = computed(() => {
  return transferablePlaylists.value.find((playlist) => playlist.identifiers.provider_id === targetPlaylistId.value);
});

watch(providers, (newProviders) => {
  if (!sourceProviderName.value && newProviders.length > 0) {
    sourceProviderName.value = newProviders[0].provider_name;
  }

  if (!targetProviderName.value && newProviders.length > 0) {
    targetProviderName.value = newProviders[0].provider_name;
  }
}, {
  immediate: true
});

watch([sourceProviderChoice, playlists], filterOnlyPlaylistsFromSourceProvider, {
  immediate: true
});

watch(sourceProviderChoice, (newValue, oldValue) => {
  if (newValue !== oldValue && sourcePlaylist.value?.meta.provider_name !== newValue) {
    sourcePlaylistId.value = undefined;
  }
})

watch([targetProviderChoice, playlists], filterOnlyPlaylistsFromTargetProvider, {
  immediate: true
});

watch(targetProviderChoice, (newValue, oldValue) => {
  if (newValue !== oldValue && targetPlaylist.value?.meta.provider_name !== newValue) {
    targetPlaylistId.value = undefined;
  }
})

onMounted(fetchLibraryPlaylists);

const onSubmit = async () => {
  if (!sourceProviderChoice.value || !targetProviderChoice.value || !sourcePlaylist.value) {
    return;
  }

  isLoading.value = true;

  try {
    let _targetPlaylistId = targetPlaylistId.value;
    if (targetPlaylistId.value && decode_entity_id(targetPlaylistId.value).provider_id === "client#create_new") {
      _targetPlaylistId = undefined;
    }

    const response = await tasksApi.syncPlaylist({
      from_provider: sourceProviderChoice.value.provider_name,
      to_provider: targetProviderChoice.value.provider_name,
      from_playlist: sourcePlaylist.value?.identifiers.provider_id,
      kind: TaskKind.PlaylistSync,
      is_dry_run: isDryRun.value,
      to_playlist: _targetPlaylistId,
    });

    emit('submitted', response.data);
    sourcePlaylistId.value = undefined;
  } finally {
    isLoading.value = false;
  }
}
</script>

<template>
  <form @submit.prevent="onSubmit">
    <AppFormSpacer direction="col">
      <AppFormSpacer direction="row">
        <AppGroupWrapper class="flex flex-col flex-1 gap-2">
          <template #label>Source</template>
          <template #description>Select the playlist to use as reference.</template>
          <template #content>
            <div class="flex gap-2 items-center">
              <ProviderSelector :providers :compact="true" v-model="sourceProviderName" />
              <PlaylistSelector :playlists="transferablePlaylists" :disabled="isLoading" v-model="sourcePlaylistId" :is-loading />
            </div>
          </template>
        </AppGroupWrapper>
        <Icon icon="material-symbols-light:arrow-right-alt-rounded" class="block text-3xl h-full my-auto text-zinc-400" />
        <AppGroupWrapper class="flex flex-col flex-1 gap-2">
          <template #label>Target</template>
          <template #description>Select the playlist to sync to or create a new one.</template>
          <template #content>
            <div class="flex gap-2 items-center">
              <ProviderSelector :providers :compact="true" v-model="targetProviderName" />
              <PlaylistSelector :playlists="targetablePlaylists" :disabled="isLoading" v-model="targetPlaylistId" :is-loading />
            </div>
          </template>
        </AppGroupWrapper>
      </AppFormSpacer>
      <div class="flex">
        <div class="w-fit">
            <AppCheckbox id="playlist-task-dry-run" v-model="isDryRun" label="Dry run" description="Runs the task as usual, but doesn't actually modify anything." :disabled="isLoading" />
        </div>
        <AppButton type="submit" :disabled="disallowSubmit || isLoading" class="min-w-30 ms-auto">
          <Icon icon="svg-spinners:3-dots-bounce" class="inline-block text-2xl" v-if="isLoading"/>
          <template v-else>
            <template v-if="targetPlaylist">Sync</template>
            <template v-else>Transfer</template>
          </template>
        </AppButton>
      </div>
    </AppFormSpacer>
  </form>
</template>
