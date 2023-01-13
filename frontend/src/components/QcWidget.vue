<script setup>
import { inject, ref, computed, watch, onMounted } from "vue";
import { Check, Close } from '@element-plus/icons-vue';
import { ElMessage } from "element-plus";
import LangQc from "../utils/langqc.js";
import { useWellStore } from '@/stores/focusWell.js';

const appConfig = inject('config');
const focusWell = useWellStore();

let client = new LangQc();

// Vars representing the current state of the form until submission
// updates the focusWell store and the backend API
let widgetQcSetting = ref(null);
let widgetFinality = ref(false);

// When the selected QC view changes, we need to reset this component
// to whatever is now in focusWell
watch(focusWell, () => {
    syncWidgetToQcState();
});

onMounted(() => {
    syncWidgetToQcState();
});

// Nicer than using onMounted() trigger?
const options = computed(() => {
    let opts = [];
    for (const setting of appConfig.value.qc_states) {
        opts.push({
            'value': setting.description,
            'label': setting.description
        });
    }
    return opts;
});

function syncWidgetToQcState() {
    if (focusWell.hasQcState) {
        widgetQcSetting.value = focusWell.getQcValue;
        widgetFinality.value = focusWell.getFinality;
    }
}

function submitQcState() {
    let [ name, well ] = focusWell.getRunAndLabel;

    client.setWellQcState(name, well, widgetQcSetting.value, widgetFinality.value)
    .then(
        (response) => {
            focusWell.updateWellQcState(response);
            // Set new "old" settings for the widgets
            syncWidgetToQcState();
            ElMessage({
                message: `Changed QC state to ${widgetQcSetting.value} : ${widgetFinality.value? "Final": "Preliminary"}`,
                type: 'success',
                duration: 3000
            });
        }
    ).catch(
        (error) => {
            ElMessage({
                message: error.message,
                type: 'error',
                duration: 5000
            })
        }
    )
}
</script>

<template>
    <div :data-testId="'previousSetting'" class="item"
        v-if="focusWell.hasQcState">
        Current QC state: {{focusWell.getFinality ? "Final": "Preliminary"}} "{{focusWell.getQcValue}}" set by "{{focusWell.getQcState.user}}"
    </div>
    <div :data-testId="'notHere'" v-else class="item">No QC setting</div>
    <el-select
        v-model="widgetQcSetting"
        :placeholder="widgetQcSetting"
        :disabled="focusWell.hasQcState ? false : true"
        :data-testId="'QC state selector'"
        class="item"
    >
        <el-option
            v-for="item in options"
            :key="item.value"
            :label="item.label"
            :value="item.value"
        />
    </el-select>
    <div class="item">
    Final: <el-switch
        v-model="widgetFinality"
        :active-icon="Check"
        :inactive-icon="Close"
        inline-prompt
        size="large"
        style="--el-switch-off-color: #131313"
        :data-testId="'QC finality selector'"
        :disabled="focusWell.hasQcState ? false : true"
    />
    </div>
    <el-button
        type="primary"
        @click="submitQcState"
        :data-testId="'QC submit'"
        :disabled="focusWell.hasQcState ? false : true"
        class="item"
    >Submit</el-button>
</template>

<style>
.item {
    padding: 5px;
}
</style>