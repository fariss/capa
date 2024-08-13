<script setup>
import { ref, computed, onMounted } from "vue";
import DescriptionPanel from "@/components/DescriptionPanel.vue";
import UploadOptions from "@/components/UploadOptions.vue";
import MetadataPanel from "@/components/MetadataPanel.vue";
import RuleMatchesTable from "@/components/RuleMatchesTable.vue";
import FunctionCapabilities from "@/components/FunctionCapabilities.vue";
import ProcessCapabilities from "@/components/ProcessCapabilities.vue";
import SettingsPanel from "@/components/SettingsPanel.vue";
import NamespaceChart from "@/components/NamespaceChart.vue";
import Toast from "primevue/toast";

import { useRdocLoader } from "@/composables/useRdocLoader";
const { rdocData, isValidVersion, loadRdoc } = useRdocLoader();

import { isGzipped, decompressGzip, readFileAsText } from "@/utils/fileUtils";

// check if we're in bundle mode by reading the VITE_IS_BUNDLE environment variable
// this variable is set to 'true' when building the release
const isBundle = import.meta.env.VITE_IS_BUNDLE === "true";

const demoRdocStatic = ref(null);
const demoRdocDynamic = ref(null);

const showCapabilitiesByFunctionOrProcess = ref(false);
const showLibraryRules = ref(false);
const showNamespaceChart = ref(false);
const showColumnFilters = ref(false);

const libraryRuleMatchesCount = computed(() => {
    if (!rdocData.value || !rdocData.value.rules) return 0;
    return Object.values(rdocData.value.rules).filter((rule) => rule.meta.lib).length;
});

const updateShowCapabilitiesByFunctionOrProcess = (value) => {
    showCapabilitiesByFunctionOrProcess.value = value;
};

const updateShowLibraryRules = (value) => {
    showLibraryRules.value = value;
};

const updateShowNamespaceChart = (value) => {
    showNamespaceChart.value = value;
};

const updateShowColumnFilters = (value) => {
    showColumnFilters.value = value;
};

const loadFromLocal = async (event) => {
    const file = event.files[0];

    let fileContent;
    if (await isGzipped(file)) {
        fileContent = await decompressGzip(file);
    } else {
        fileContent = await readFileAsText(file);
    }

    const jsonData = JSON.parse(fileContent);

    loadRdoc(jsonData);
};

const loadFromURL = (url) => {
    loadRdoc(url);
};

const loadDemoDataStatic = () => {
    if (demoRdocStatic.value) {
        loadRdoc(demoRdocStatic.value);
    }
};

const loadDemoDataDynamic = () => {
    if (demoRdocDynamic.value) {
        loadRdoc(demoRdocDynamic.value);
    }
};

onMounted(async () => {
    // Clear out sessionStorage to prevent stale data from being used
    sessionStorage.clear();

    // Check if the URL contains a rdoc parameter and load the data from that URL
    const urlParams = new URLSearchParams(window.location.search);
    const encodedRdocURL = urlParams.get("rdoc");
    if (encodedRdocURL) {
        const rdocURL = decodeURIComponent(encodedRdocURL);
        loadFromURL(rdocURL);
    }

    // Load demo data if not in bundle mode
    if (!isBundle) {
        try {
            // Import static demo data
            await import("@testfiles/rd/al-khaser_x64.exe_.json").then((module) => {
                demoRdocStatic.value = module.default;
            });

            await import("@testfiles/rd/0000a65749f5902c4d82ffa701198038f0b4870b00a27cfca109f8f933476d82.json").then(
                (module) => {
                    demoRdocDynamic.value = module.default;
                }
            );
        } catch (error) {
            console.error("Error importing demo files:", error);
        }
    }
});
</script>

<template>
    <Panel v-if="!rdocData || !isValidVersion">
        <DescriptionPanel />
        <UploadOptions
            @load-from-local="loadFromLocal"
            @load-from-url="loadFromURL"
            @load-demo-static="loadDemoDataStatic"
            @load-demo-dynamic="loadDemoDataDynamic"
        />
    </Panel>

    <Toast position="bottom-center" group="bc" />
    <template v-if="rdocData && isValidVersion">
        <MetadataPanel :data="rdocData" />
        <SettingsPanel
            :flavor="rdocData.meta.flavor"
            :library-rule-matches-count="libraryRuleMatchesCount"
            @update:show-capabilities-by-function-or-process="updateShowCapabilitiesByFunctionOrProcess"
            @update:show-library-rules="updateShowLibraryRules"
            @update:show-namespace-chart="updateShowNamespaceChart"
            @update:show-column-filters="updateShowColumnFilters"
        />

        <RuleMatchesTable
            v-if="!showCapabilitiesByFunctionOrProcess && !showNamespaceChart"
            :data="rdocData"
            :show-library-rules="showLibraryRules"
            :show-column-filters="showColumnFilters"
        />
        <FunctionCapabilities
            v-if="rdocData.meta.flavor === 'static' && showCapabilitiesByFunctionOrProcess && !showNamespaceChart"
            :data="rdocData"
            :show-library-rules="showLibraryRules"
        />
        <ProcessCapabilities
            v-else-if="rdocData.meta.flavor === 'dynamic' && showCapabilitiesByFunctionOrProcess && !showNamespaceChart"
            :data="rdocData"
            :show-capabilities-by-process="showCapabilitiesByFunctionOrProcess"
            :show-library-rules="showLibraryRules"
        />
        <NamespaceChart v-else-if="showNamespaceChart" :data="rdocData" />
    </template>
</template>
