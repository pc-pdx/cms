<template>

    <node-view-wrapper>
        <div
            class="bard-inline-image-container shadow-sm"
            :class="{
                'border-blue-400' : selected,
            }"
        >
            <div v-if="src" class="p-2 text-center" draggable="true" data-drag-handle>
                <div ref="content" hidden />
                <img :src="src" class="block mx-auto rounded-sm" />
            </div>

            <div class="@container/toolbar flex items-center border-t dark:border-dark-900 justify-center py-2 px-2 text-2xs text-white dark:text-dark-150 text-center space-x-1 sm:space-x-3">
                <button v-if="!src" @click="openSelector" type="button" class="flex btn btn-sm px-3 py-1.5">
                    <svg-icon name="folder-image" class="h-4" />
                    <span class="rtl:mr-2 ltr:ml-2 hidden @md/toolbar:inline-block">{{ __('Choose Image') }}</span>
                </button>
                <button v-if="src" @click="edit" type="button" class="flex btn btn-sm px-3 py-1.5">
                    <svg-icon name="pencil" class="h-4" />
                    <span class="rtl:mr-2 ltr:ml-2 hidden @md/toolbar:inline-block">{{ __('Edit Image') }}</span>
                </button>
                <button v-if="src" @click="toggleAltEditor" type="button" class="flex btn btn-sm px-3 py-1.5" :class="{ active: showingAltEdit }">
                    <svg-icon name="rename-file" class="h-4" />
                    <span class="rtl:mr-2 ltr:ml-2 hidden @md/toolbar:inline-block">{{ __('Override Alt') }}</span>
                </button>
                <button v-if="src" @click="openSelector" type="button" class="flex btn btn-sm px-3 py-1.5">
                    <svg-icon name="swap" class="h-4" />
                    <span class="rtl:mr-2 ltr:ml-2 hidden @md/toolbar:inline-block">{{ __('Replace') }}</span>
                </button>
                <button @click="deleteNode" class="flex btn btn-sm text-red-500 px-3 py-1.5">
                    <svg-icon name="trash" class="h-4" />
                    <span class="rtl:mr-2 ltr:ml-2 hidden @md/toolbar:inline-block">{{ __('Remove') }}</span>
                </button>
            </div>

            <div v-if="showingAltEdit" class="flex items-center p-2 border-t dark:border-dark-900 rounded-b" @paste.stop>
                <text-input name="alt" :focus="showingAltEdit" v-model="alt" :placeholder="assetAlt" :prepend="__('Alt Text')" class="flex-1" />
            </div>

            <stack
                v-if="showingSelector"
                name="asset-selector"
                @closed="closeSelector"
            >
                <selector
                    :container="extension.options.bard.config.container"
                    :folder="extension.options.bard.config.folder || '/'"
                    :restrict-container-navigation="true"
                    :restrict-folder-navigation="extension.options.bard.config.restrict_assets"
                    :selected="selections"
                    :view-mode="'grid'"
                    :max-files="1"
                    @selected="assetsSelected"
                    @closed="closeSelector">
                </selector>
            </stack>

            <asset-editor
                v-if="editing"
                :id="assetId"
                :showToolbar="false"
                :allow-deleting="false"
                @closed="closeEditor"
                @saved="editorAssetSaved"
                @actionCompleted="actionCompleted"
            >
            </asset-editor>
        </div>
    </node-view-wrapper>

</template>

<script>
import Asset from '../assets/Asset';
import { NodeViewWrapper } from '@tiptap/vue-2';
import Selector from '../../assets/Selector.vue';

export default {
    mixins: [Asset],

    components: {
        NodeViewWrapper,
        Selector,
    },

    inject: ['storeName'],

    props: [
        'editor', // the editor instance
        'node', // access the current node
        'decorations', // an array of decorations
        'selected', // true when there is a NodeSelection at the current node view
        'extension', // access to the node extension, for example to get options
        'getPos', // get the document position of the current node
        'updateAttributes', // update attributes of the current node.
        'deleteNode', // delete the current node
    ],

    data() {
        return {
            assetId: null,
            assetAlt: null,
            editorAsset: null,
            showingSelector: false,
            loading: false,
            alt: this.node.attrs.alt,
            showingAltEdit: !!this.node.attrs.alt,
        }
    },

    computed: {

        src() {
            if (this.editorAsset) {
                return this.editorAsset.url;
            }
        },

        actualSrc() {
            if (this.editorAsset) {
                return `asset::${this.assetId}`;
            }

            return this.src;
        },

        selections() {
            return this.assetId ? [this.assetId] : [];
        }

    },

    created() {
        let src = this.node.attrs.src;

        if (this.node.isNew) {
            this.openSelector();
        }

        if (src && src.startsWith('asset:')) {
            this.assetId = src.substr(7);
        }

        let id = this.assetId || src;
        if (id) this.loadAsset(id);
    },

    watch: {

        actualSrc(src) {
            if ( !this.node.attrs.src) {
                this.updateAttributes({ src, asset: !!this.assetId });
            }
        },

        alt(alt) {
            this.updateAttributes({ alt });
        }

    },

    methods: {

        openSelector() {
            this.showingSelector = true;
        },

        closeSelector() {
            this.showingSelector = false;
        },

        assetsSelected(selections) {
            this.loading = true;
            this.assetId = selections[0];
            this.loadAsset(this.assetId);
        },

        loadAsset(id) {
            let preloaded = _.find(this.$store.state.publish[this.storeName].preloadedAssets, asset => asset.id === id);

            if (preloaded) {
                // TODO
                // Disabling preloading temporarily. It's causing an infinite loop.
                // It wasn't working on 3.2 anyway. It wasn't preloading, the AJAX request was always happening.
                // this.setAsset(preloaded);
                // return;
            }

            this.$axios.post(cp_url('assets-fieldtype'), {
                assets: [id],
            }).then(response => {
                this.setAsset(response.data[0]);
            });
        },

        setAsset(asset) {
            this.editorAsset = asset;
            this.assetId = asset.id;
            this.assetAlt = asset.values.alt;
            this.loading = false;
            this.updateAttributes({ src: this.actualSrc });
        },

        toggleAltEditor() {
            this.showingAltEdit = !this.showingAltEdit;
            if (!this.showingAltEdit) {
                this.alt = null;
            }
        },

        editorAssetSaved(asset) {
            this.setAsset(asset);
            this.closeEditor();
        },
    },

    updated() {
        // This is a workaround to avoid Firefox's inability to select inputs/textareas when the
        // parent element is set to draggable: https://bugzilla.mozilla.org/show_bug.cgi?id=739071
        this.$el.setAttribute('draggable', false);
    }
}
</script>
