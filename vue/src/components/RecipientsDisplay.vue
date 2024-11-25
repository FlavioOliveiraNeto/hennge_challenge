<script setup lang="ts"></script>

<template>
    <div class="recipients-display" ref="recipientsContainer">
      <div
        class="recipients-list"
        ref="recipientsList"
        :style="{ display: 'flex', alignItems: 'center' }"
      >
        <!-- Destinatários visíveis -->
        <span v-for="(recipient, index) in visibleRecipients" :key="index">
          {{ recipient }}<span v-if="index < visibleRecipients.length - 1">,&nbsp</span>
        </span>
  
        <!-- Reticências e Badge quando houver truncamento -->
        <span v-if="hasTrimmedRecipients">&nbsp</span>
        <RecipientsBadge
          ref="recipientsBadge"
          class="badge"
          v-if="hasTrimmedRecipients"
          :num-truncated="trimmedCount"
          @mouseover="showTooltip = true"
          @mouseleave="showTooltip = false"
        />
      </div>
  
      <div
        class="recipients-tooltip"
        ref="recipientsTooltip"
        :class="{ 'tooltip--column': tooltipColumnMode }"
        v-if="showTooltip"
      >
        <div v-for="(recipient, index) in recipients" :key="index">
          <span v-if="index + 1 == recipients.length">{{ recipient }}</span>
          <span v-else>{{ recipient }},&nbsp;</span>
        </div>
      </div>
    </div>
  </template>
  
<script lang="ts">
import { defineComponent, ref, computed, onMounted, watch, nextTick } from "vue";
import RecipientsBadge from "@/components/RecipientsBadge.vue";

export default defineComponent({
  name: "RecipientsDisplay",
  components: { RecipientsBadge },
  props: {
    recipients: {
      type: Array as () => string[],
      required: true,
    },
  },
  setup(props) {
    const recipientsContainer = ref<HTMLDivElement | null>(null);
    const recipientsBadge = ref<HTMLDivElement | null>(null);
    const recipientsList = ref<HTMLDivElement | null>(null);
    const recipientsTooltip = ref<HTMLDivElement | null>(null);

    const visibleRecipients = ref<string[]>([]);
    const trimmedCount = ref<number>(0);
    const showTooltip = ref<boolean>(false);
    const tooltipColumnMode = ref<boolean>(false);

    const hasTrimmedRecipients = computed(() => trimmedCount.value > 0);

    const getElementWidth = (text: string): number => {
      const tempSpan = document.createElement("span");
      tempSpan.innerText = text;
      tempSpan.style.visibility = "hidden";
      tempSpan.style.display = "inline-block";

      recipientsList.value?.appendChild(tempSpan);
      const width = tempSpan.offsetWidth;
      recipientsList.value?.removeChild(tempSpan);

      return width;
    };

    const truncateRecipient = (text: string, maxWidth: number): string => {
      let truncated = text;
      while (getElementWidth(truncated + "...") > maxWidth && truncated.length > 1) {
        truncated = truncated.slice(0, -1);
      }
      return truncated.length <= 3 ? "..." : truncated + "...";
    };

    const calculateVisibleRecipients = async (): Promise<void> => {
        if (!recipientsContainer.value || !recipientsList.value) return;

        await nextTick(); // Aguarda renderização do badge

        const containerWidth = recipientsContainer.value.offsetWidth;
        const badgeWidth = recipientsBadge.value?.$el?.offsetWidth || 0;
        const availableWidth = containerWidth - badgeWidth - 60;

        visibleRecipients.value = [];
        trimmedCount.value = 0;

        let totalWidth = 0;

        // Variável para saber se o primeiro destinatário foi truncado
        let firstRecipientTruncated = false;

        for (const recipient of props.recipients) {
            const recipientWidth = getElementWidth(recipient);

            if (totalWidth + recipientWidth <= availableWidth) {
                visibleRecipients.value.push(recipient);
                totalWidth += recipientWidth + 8; // 8 é o espaço extra entre os elementos
            } else if (visibleRecipients.value.length === 0) {
                // Quando o primeiro destinatário não couber
                const truncatedRecipient = truncateRecipient(recipient, availableWidth);
                if (truncatedRecipient === "...") {
                    visibleRecipients.value = ['...']; // Mostra apenas '...'
                    firstRecipientTruncated = true;
                    break; // Não continua, pois já foi truncado
                } else {
                    visibleRecipients.value.push(truncatedRecipient);
                    break; // Se não for truncado, também para aqui
                }
            } else {
                break;
            }
        }

        trimmedCount.value = props.recipients.length - visibleRecipients.value.length;

        // Adiciona +1 à contagem caso o primeiro destinatário tenha sido truncado para '...'
        if (firstRecipientTruncated) {
        trimmedCount.value += 1;
        }
    };

    const adjustTooltipPosition = (): void => {
      if (recipientsTooltip.value) {
        const tooltipRect = recipientsTooltip.value.getBoundingClientRect();
        const fitsVertically =
          tooltipRect.top >= 0 &&
          tooltipRect.bottom <= window.innerHeight;
        const fitsHorizontally =
          tooltipRect.left >= 0 &&
          tooltipRect.right <= window.innerWidth;

        tooltipColumnMode.value = !(fitsVertically && fitsHorizontally);
      }
    };

    const handleResize = (): void => {
      calculateVisibleRecipients();
      adjustTooltipPosition();
    };

    onMounted(() => {
      calculateVisibleRecipients();
      window.addEventListener("resize", handleResize);
    });

    watch(
      () => showTooltip.value,
      (newValue) => {
        if (newValue) {
          nextTick(() => adjustTooltipPosition());
        }
      }
    );

    return {
      recipientsContainer,
      recipientsBadge,
      recipientsList,
      recipientsTooltip,
      visibleRecipients,
      trimmedCount,
      showTooltip,
      hasTrimmedRecipients,
      tooltipColumnMode,
    };
  },
});
</script>
  
<style scoped>
.recipients-display {
  position: relative;
  display: inline;
}

.recipients-list {
  display: flex;
  align-items: center;
  flex-wrap: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;

  span.badge {
    margin-inline-start: auto;
  }
}

.recipients-tooltip {
  margin: 8px;
  padding: 8px 16px;
  background-color: #666;
  color: #f0f0f0;
  border-radius: 8px;
  display: flex;
  align-items: center;
  position: fixed;
  top: 8px;
  right: 8px;
  white-space: nowrap;
  transition: flex-direction 0.2s ease;
}

.recipients-tooltip.tooltip--column {
  flex-direction: column;
  align-items: flex-start;
  white-space: normal;
}
</style>