<script setup>
import { computed, ref, watch } from 'vue';
import { useI18n } from '@/composables/useI18n';
import { shorten, getChoiceString, explorerUrl } from '@/helpers/utils';
import { useClient } from '@/composables/useClient';
import { useIntl } from '@/composables/useIntl';
import { getPower } from '@/helpers/snapshot';
import { useWeb3 } from '@/composables/useWeb3';
import { useProposals } from '@/composables';

const { web3Account } = useWeb3();

const vp = ref(0);
const vpByStrategy = ref([]);
const vpLoading = ref(false);
const vpLoadingFailed = ref(false);
const vpLoaded = ref(false);
const reason = ref('');

const props = defineProps({
  open: Boolean,
  space: Object,
  proposal: Object,
  selectedChoices: [Object, Number],
  snapshot: String,
  strategies: Object
});

const emit = defineEmits(['reload', 'close', 'openPostVoteModal']);

const { t } = useI18n();
const { send, isSending } = useClient();
const format = getChoiceString;
const { formatNumber, formatCompactNumber } = useIntl();
const { addVotedProposalId } = useProposals();

const symbols = computed(() =>
  props.strategies.map(strategy => strategy.params.symbol || '')
);

async function handleSubmit() {
  const result = await send(props.space, 'vote', {
    proposal: props.proposal,
    choice: props.selectedChoices,
    reason: reason.value
  });
  console.log('Result', result);
  if (result.id) {
    emit('openPostVoteModal');
    emit('reload');
    addVotedProposalId(props.proposal.id);
    emit('close');
  }
}

watch(
  () => [props.open, web3Account.value],
  async () => {
    if (props.open === false) return;
    vpLoading.value = true;
    vpLoadingFailed.value = false;
    try {
      const result = await getPower(
        props.space,
        web3Account.value,
        props.proposal
      );
      vp.value = result.vp;
      vpByStrategy.value = result.vp_by_strategy;
    } catch (e) {
      vpLoadingFailed.value = true;
      console.log(e);
    } finally {
      vpLoaded.value = true;
      vpLoading.value = false;
    }
  }
);
</script>

<template>
  <BaseModal
    :open="open"
    :hide-close="true"
    class="flex"
    @close="$emit('close')"
  >
    <div class="flex flex-auto flex-col">
      <h4 class="m-4 mb-0 text-center">
        {{ $tc('proposal.castVote') }}
      </h4>
      <div slim class="m-4 text-skin-link">
        <div class="flex">
          <span class="mr-1 flex-auto text-skin-text" v-text="$t('options')" />
          <span
            v-tippy="{
              content:
                format(proposal, selectedChoices).length > 30
                  ? format(proposal, selectedChoices)
                  : null
            }"
            class="ml-4 truncate text-right"
          >
            {{ format(proposal, selectedChoices) }}
          </span>
        </div>
        <div class="flex">
          <span class="mr-1 flex-auto text-skin-text" v-text="$t('snapshot')" />
          <BaseLink
            :link="explorerUrl(proposal.network, proposal.snapshot, 'block')"
            class="float-right"
          >
            {{ formatNumber(proposal.snapshot) }}
          </BaseLink>
        </div>
        <div class="flex">
          <span
            class="mr-1 flex-auto text-skin-text"
            v-text="$t('votingPower')"
          />
          <span v-if="vpLoadingFailed" class="item-center flex">
            <BaseIcon name="warning" size="22" class="text-red" />
          </span>
          <span
            v-else-if="vpLoaded && !vpLoading"
            v-tippy="{
              content: vpByStrategy
                .map(
                  (score, index) =>
                    `${formatCompactNumber(score)} ${symbols[index]}`
                )
                .join(' + ')
            }"
          >
            {{ formatCompactNumber(vp) }}
            {{ shorten(proposal.symbol || space.symbol, 'symbol') }}
          </span>
          <LoadingSpinner v-else />
          <BaseLink
            v-if="vp === 0 && vpLoaded && !vpLoading && !vpLoadingFailed"
            link="https://github.com/snapshot-labs/snapshot/discussions/767#discussioncomment-1400614"
            class="ml-1 flex items-center"
          >
            <BaseIcon name="info" size="24" class="text-skin-text" />
          </BaseLink>
        </div>
        <div class="mt-3 flex">
          <TextareaAutosize
            v-model="reason"
            :max-length="140"
            class="s-input !rounded-3xl"
            :placeholder="$t('comment.placeholder')"
          />
        </div>
        <div v-if="vpLoadingFailed" class="mt-3">{{ t('vpError') }}</div>
      </div>
    </div>
    <template #footer>
      <div class="float-left w-2/4 pr-2">
        <BaseButton type="button" class="w-full" @click="$emit('close')">
          {{ $t('cancel') }}
        </BaseButton>
      </div>
      <div class="float-left w-2/4 pl-2">
        <BaseButton
          :disabled="vp === 0 || isSending"
          :loading="isSending"
          type="submit"
          class="w-full"
          primary
          @click="handleSubmit"
        >
          {{ $t('confirm') }}
        </BaseButton>
      </div>
    </template>
  </BaseModal>
</template>
