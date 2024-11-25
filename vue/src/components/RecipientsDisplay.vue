<script setup lang="ts"></script>

<template>
    <div class="recipients-display" ref="recipientsContainer">
        <div
            class="recipients-list"
            ref="recipientsList"
            :style="{ display: 'flex', alignItems: 'center' }"
        >
            <!-- Exibe o primeiro email, truncado se necessário -->
            <span v-if="visibleRecipients.length > 0">
                {{ visibleRecipients[0] }}
            </span>

            <!-- Exibe o "+N" quando há emails truncados -->
            <span v-if="hasTrimmedRecipients">, ... </span>
            <!-- Componente RecipientsBadge -->
            <RecipientsBadge
                ref="recipientsBadge"
                v-if="hasTrimmedRecipients"
                :num-truncated="trimmedCount"
                @mouseover="showTooltip = true"
                @mouseleave="showTooltip = false"
            />
        </div>
        <!-- Tooltip para exibir a lista completa -->
        <div
            class="recipients-tooltip"
            v-if="showTooltip"
            :style="tooltipStyle"
        >
            {{ recipients.join(', ') }}
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
        const recipientsList = ref<HTMLDivElement | null>(null);
        const visibleRecipients = ref<string[]>([]);
        const trimmedCount = ref<number>(0);
        const showTooltip = ref<boolean>(false);

        const tooltipStyle = computed(() => ({
            margin: "8px",
            padding: "8px 16px",
            backgroundColor: "#666",
            color: "#f0f0f0",
            borderRadius: "8px",
            display: "flex",
            alignItems: "center",
            position: "fixed",
            top: "8px",
            right: "8px",
            whiteSpace: "nowrap",
        }));

        const calculateVisibleRecipients = async (): Promise<void> => {
            if (!recipientsContainer.value || !recipientsList.value) return;

            const containerWidth = recipientsContainer.value.offsetWidth;

            // Reservar espaço para "+N"
            const badgeWidth = 40; // Largura aproximada de "+N"
            const availableWidth = containerWidth - badgeWidth - 16; // Reserva 16px para margens

            let totalWidth = 0;

            visibleRecipients.value = [];
            trimmedCount.value = 0;

            // Verificar o primeiro email
            if (props.recipients.length > 0) {
                const firstRecipient = props.recipients[0];
                const tempSpan = document.createElement("span");
                tempSpan.innerText = firstRecipient;
                tempSpan.style.display = "inline";
                tempSpan.style.visibility = "hidden";
                recipientsList.value?.appendChild(tempSpan);

                let firstRecipientWidth = tempSpan.offsetWidth;
                recipientsList.value?.removeChild(tempSpan);

                // Truncar se o primeiro email não couber
                if (firstRecipientWidth > availableWidth) {
                    let truncatedEmail = firstRecipient;
                    while (
                        truncatedEmail.length > 1 &&
                        firstRecipientWidth > availableWidth
                    ) {
                        truncatedEmail = truncatedEmail.slice(0, -1);
                        tempSpan.innerText = truncatedEmail + "...";
                        recipientsList.value?.appendChild(tempSpan);
                        firstRecipientWidth = tempSpan.offsetWidth;
                        recipientsList.value?.removeChild(tempSpan);
                    }
                    visibleRecipients.value.push(truncatedEmail + "...");
                } else {
                    visibleRecipients.value.push(firstRecipient);
                }

                totalWidth = firstRecipientWidth;
            }

            // Calcular o número de emails truncados
            trimmedCount.value = props.recipients.length - 1;
        };

        onMounted(() => {
            calculateVisibleRecipients();
            window.addEventListener("resize", calculateVisibleRecipients);
        });

        watch(
            () => props.recipients,
            () => {
                nextTick(() => {
                    calculateVisibleRecipients();
                });
            }
        );

        return {
            recipientsContainer,
            recipientsList,
            visibleRecipients,
            trimmedCount,
            showTooltip,
            tooltipStyle,
            hasTrimmedRecipients: computed(() => trimmedCount.value > 0),
        };
    },
});
</script>

<style scoped>
.recipients-list {
    display: flex;
    align-items: center;
    flex-wrap: nowrap;
    overflow: hidden;
}

.recipients-tooltip {
    white-space: nowrap;
}

.recipients-display {
    position: relative;
    display: flex;
    align-items: center;
}
</style>
