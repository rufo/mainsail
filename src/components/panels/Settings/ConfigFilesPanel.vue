<style>
    .my-editor {
        background: #2d2d2d;
        color: #ccc;
        font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
        font-size: 14px;
        line-height: 1.5;
        padding: 5px;
    }

    .prism-editor__textarea:focus {
        outline: none;
    }

    .config-editor-overlay div.v-card {
        position: relative;
    }

    .config-editor-overlay div.v-card header {
        position: sticky;
        top: 0;
        width: 100%;
        z-index: 1;
    }
</style>

<template>
    <v-row>
        <v-col>
            <v-card>
                <v-card-title>
                    Config Files
                    <v-spacer class="d-none d-sm-block"></v-spacer>
                    <input type="file" ref="fileUpload" style="display: none" multiple @change="uploadFile" />
                    <v-item-group class="v-btn-toggle my-5 my-sm-0 col-12 col-sm-auto px-0 py-0">
                        <v-btn v-if="currentPath !== '' && currentPath !== '/config_examples'" class="flex-grow-1" @click="uploadFileButton" :loading="loadings.includes['configFileUpload']"><v-icon>mdi-file-upload</v-icon></v-btn>
                        <v-btn v-if="currentPath !== '' && currentPath !== '/config_examples'" class="flex-grow-1" @click="createFile"><v-icon>mdi-file-plus</v-icon></v-btn>
                        <v-btn v-if="currentPath !== '' && currentPath !== '/config_examples'" class="flex-grow-1" @click="createFolder"><v-icon>mdi-folder-plus</v-icon></v-btn>
                        <v-btn class="flex-grow-1" @click="refreshFileList"><v-icon>mdi-refresh</v-icon></v-btn>
                        <v-menu :offset-y="true" title="Setup current list">
                            <template v-slot:activator="{ on, attrs }">
                                <v-btn class="flex-grow-1" v-bind="attrs" v-on="on"><v-icon class="">mdi-cog</v-icon></v-btn>
                            </template>
                            <v-list>
                                <v-list-item class="minHeight36">
                                    <v-checkbox class="mt-0" hide-details v-model="showHiddenFiles" label="Hidden files"></v-checkbox>
                                </v-list-item>
                            </v-list>
                        </v-menu>
                    </v-item-group>
                </v-card-title>
                <v-card-subtitle>Current path: {{ this.currentPath === "" ? "/" : this.currentPath }}</v-card-subtitle>
                <v-data-table
                    :items="files"
                    class="files-table"
                    :headers="headers"
                    :options="options"
                    :page.sync="currentPage"
                    :custom-sort="sortFiles"
                    :sort-by.sync="sortBy"
                    :sort-desc.sync="sortDesc"
                    :items-per-page.sync="countPerPage"
                    :footer-props="{
                        itemsPerPageText: 'Files'
                    }"
                    mobile-breakpoint="0"
                    item-key="name">

                    <template #no-data>
                        <div class="text-center">empty</div>
                    </template>

                    <template slot="body.prepend" v-if="(currentPath !== '')">
                       <tr
                            class="file-list-cursor"
                            @click="clickRowGoBack">
                            <td class="pr-0 text-center" style="width: 32px;"><v-icon>mdi-folder-upload</v-icon></td>
                            <td class=" " colspan="8">..</td>
                        </tr>
                    </template>

                    <template #item="{ item }">
                        <tr
                            @contextmenu="showContextMenu($event, item)"
                            @click="clickRow(item)"
                            class="file-list-cursor"
                            :data-name="item.filename">
                            <td class="pr-0 text-center" style="width: 32px;">
                                <v-icon v-if="item.isDirectory">mdi-folder</v-icon>
                                <v-icon v-if="!item.isDirectory">mdi-file</v-icon>
                            </td>
                            <td class=" ">{{ item.filename }}</td>
                            <td class="text-no-wrap text-right">{{ item.isDirectory ? '--' : formatFilesize(item.size) }}</td>
                            <td class="text-right">{{ formatDate(item.modified) }}</td>
                        </tr>
                    </template>
                </v-data-table>
            </v-card>
            <v-menu v-model="contextMenu.shown" :position-x="contextMenu.x" :position-y="contextMenu.y" absolute offset-y>
                <v-list>
                    <v-list-item @click="clickRow(contextMenu.item)" v-if="!contextMenu.item.isDirectory">
                        <v-icon class="mr-1">mdi-file-document-edit-outline</v-icon> Edit file
                    </v-list-item>
                    <v-list-item @click="downloadFile" v-if="!contextMenu.item.isDirectory">
                        <v-icon class="mr-1">mdi-cloud-download</v-icon> Download
                    </v-list-item>
                    <v-list-item @click="renameFile(contextMenu.item)" v-if="!contextMenu.item.isDirectory && currentPath !== '/config_examples'">
                        <v-icon class="mr-1">mdi-rename-box</v-icon> Rename
                    </v-list-item>
                    <v-list-item @click="removeFile" v-if="!contextMenu.item.isDirectory && currentPath !== '/config_examples'">
                        <v-icon class="mr-1">mdi-delete</v-icon> Delete
                    </v-list-item>
                    <v-list-item @click="deleteDirectoryAction" v-if="contextMenu.item.isDirectory && currentPath !== '' && currentPath !== '/config_examples'">
                        <v-icon class="mr-1">mdi-delete</v-icon> Delete
                    </v-list-item>
                </v-list>
            </v-menu>
            <v-dialog v-model="editor.showLoader" hide-overlay persistent width="300" >
                <v-card color="primary" dark >
                    <v-card-text>
                        Please stand by
                        <v-progress-linear indeterminate color="white" class="mb-0" ></v-progress-linear>
                    </v-card-text>
                </v-card>
            </v-dialog>
            <v-dialog v-model="editor.show" fullscreen hide-overlay transition="dialog-bottom-transition" content-class="config-editor-overlay">
                <v-card d-flex>
                    <v-toolbar dark color="primary">
                        <v-btn icon dark @click="editor.show = false">
                            <v-icon>mdi-close</v-icon>
                        </v-btn>
                        <v-toolbar-title>{{ editor.item.filename }}</v-toolbar-title>
                        <v-spacer></v-spacer>
                        <v-toolbar-items>
                            <v-btn dark text href="https://www.klipper3d.org/Config_Reference.html" target="_blank" class="d-none d-md-flex"><v-icon small class="mr-1">mdi-help</v-icon>Config Reference</v-btn>
                            <v-btn dark text @click="saveFile(false)" v-if="currentPath !== '/config_examples'"><v-icon small class="mr-1">mdi-content-save</v-icon><span class="d-none d-sm-inline">Save</span></v-btn>
                            <v-btn dark text @click="saveFile(true)" v-if="currentPath !== '/config_examples' && !['printing', 'paused'].includes(printer_state)" class="d-none d-sm-flex"><v-icon small class="mr-1">mdi-restart</v-icon>Save & restart</v-btn>
                        </v-toolbar-items>
                    </v-toolbar>
                    <prism-editor class="my-editor" v-model="editor.sourcecode" :readonly="editor.readonly" :highlight="highlighter" line-numbers></prism-editor>
                </v-card>
            </v-dialog>
            <v-dialog v-model="dialogRenameFile.show" max-width="400">
                <v-card>
                    <v-card-title class="headline">Rename File</v-card-title>
                    <v-card-text>
                        <v-text-field label="Name" required v-model="dialogRenameFile.newName"></v-text-field>
                    </v-card-text>
                    <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-btn color="" text @click="dialogRenameFile.show = false">Cancel</v-btn>
                        <v-btn color="primary" text @click="renameFileAction">rename</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
            <v-dialog v-model="dialogCreateFile.show" max-width="400">
                <v-card>
                    <v-card-title class="headline">Create File</v-card-title>
                    <v-card-text>
                        <v-text-field label="Name" required v-model="dialogCreateFile.name"></v-text-field>
                    </v-card-text>
                    <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-btn color="" text @click="dialogCreateFile.show = false">Cancel</v-btn>
                        <v-btn color="primary" text @click="createFileAction">create</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
            <v-dialog v-model="dialogCreateFolder.show" max-width="400">
                <v-card>
                    <v-card-title class="headline">Create Folder</v-card-title>
                    <v-card-text>
                        <v-text-field label="Name" required v-model="dialogCreateFolder.name"></v-text-field>
                    </v-card-text>
                    <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-btn color="" text @click="dialogCreateFolder.show = false">Cancel</v-btn>
                        <v-btn color="primary" text @click="createFolderAction">create</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
            <v-snackbar
                :timeout="-1"
                :value="true"
                fixed
                right
                bottom
                dark
                v-model="uploadSnackbar.status"
            >
                <span v-if="uploadSnackbar.max > 1" class="mr-1">({{ uploadSnackbar.number }}/{{ uploadSnackbar.max }})</span><strong>Uploading {{ uploadSnackbar.filename }}</strong><br />
                {{ Math.round(uploadSnackbar.percent) }} % @ {{ formatFilesize(Math.round(uploadSnackbar.speed)) }}/s<br />
                <v-progress-linear class="mt-2" :value="uploadSnackbar.percent"></v-progress-linear>
                <template v-slot:action="{ attrs }">
                    <v-btn
                        color="red"
                        text
                        v-bind="attrs"
                        @click="cancelUpload"
                        style="min-width: auto;"
                    >
                        <v-icon class="0">mdi-close</v-icon>
                    </v-btn>
                </template>
            </v-snackbar>
        </v-col>
    </v-row>
</template>

<script>
    import { mapState } from 'vuex'
    import {findDirectory} from "@/plugins/helpers";

    import { PrismEditor } from 'vue-prism-editor';
    import 'vue-prism-editor/dist/prismeditor.min.css'; // import the styles somewhere

    // import highlighting library (you can use any library you want just return html string)
    import { highlight, languages } from 'prismjs/components/prism-core';
    import 'prismjs/components/prism-editorconfig';
    import 'prismjs/components/prism-ini';
    import 'prismjs/themes/prism-okaidia.css';
    import axios from "axios"; // import syntax highlighting styles

    export default {
        components: {
            PrismEditor,
        },
        data: function() {
            return {
                sortBy: 'filename',
                sortDesc: false,
                selected: [],
                headers: [
                    { text: '', value: '', },
                    { text: 'Name', value: 'filename', },
                    { text: 'Filesize', value: 'size', align: 'right', },
                    { text: 'Last modified', value: 'modified', align: 'right', },
                ],
                options: {
                },
                currentPage: 1,
                files: [],
                currentPath: '/config',
                editor: {
                    show: false,
                    showLoader: false,
                    readonly: false,
                    item: {
                        filename: "",
                    },
                    sourcecode: ''
                },
                contextMenu: {
                    shown: false,
                    isDirectory: false,
                    touchTimer: undefined,
                    x: 0,
                    y: 0,
                    item: {}
                },
                dialogRenameFile: {
                    show: false,
                    newName: "",
                    item: {}
                },
                dialogCreateFile: {
                    show: false,
                    name: "",
                },
                dialogCreateFolder: {
                    show: false,
                    name: "",
                },
                uploadSnackbar: {
                    status: false,
                    filename: "",
                    percent: 0,
                    speed: 0,
                    total: 0,
                    number: 0,
                    max: 0,
                    cancelTokenSource: "",
                    lastProgress: {
                        time: 0,
                        loaded: 0
                    }
                }
            }
        },
        computed: {
            ...mapState({
                filetree: state => state.files.filetree,
                hostname: state => state.socket.hostname,
                port: state => state.socket.port,
                loadings: state => state.socket.loadings,
                printer_state: state => state.printer.print_stats.state,
            }),
            countPerPage: {
                get: function() {
                    return this.$store.state.gui.settings.configfiles.countPerPage
                },
                set: function(newVal) {
                    return this.$store.dispatch("gui/setSettings", { settings: { configfiles: { countPerPage: newVal } } });
                }
            },
            showHiddenFiles: {
                get: function() {
                    return this.$store.state.gui.settings.configfiles.showHiddenFiles
                },
                set: function(newVal) {
                    return this.$store.dispatch("gui/setSettings", { settings: { configfiles: { showHiddenFiles: newVal } } })
                }
            },
        },
        created() {
            this.loadPath();
        },
        methods: {
            highlighter(code) {
                return highlight(code, languages.ini); //returns html
            },
            refreshFileList: function() {
                if (this.currentPath === "") {
                    this.$socket.sendObj('server.files.get_directory', { path: 'config' }, 'files/getDirectory');
                    this.$socket.sendObj('server.files.get_directory', { path: 'config_examples' }, 'files/getDirectory');
                } else this.$socket.sendObj('server.files.get_directory', { path: this.currentPath.substring(1) }, 'files/getDirectory');
            },
            formatDate(date) {
                let tmp2 = new Date(date);

                return tmp2.toLocaleString().replace(',', '');
            },
            formatFilesize(fileSizeInBytes) {
                let i = -1;
                let byteUnits = [' kB', ' MB', ' GB', ' TB', 'PB', 'EB', 'ZB', 'YB'];
                do {
                    fileSizeInBytes = fileSizeInBytes / 1024;
                    i++;
                } while (fileSizeInBytes > 1024);

                return Math.max(fileSizeInBytes, 0.1).toFixed(1) + byteUnits[i];
            },
            sortFiles(items, sortBy, sortDesc) {
                sortBy = sortBy.length ? sortBy[0] : 'filename';
                sortDesc = sortDesc[0];

                if (items !== undefined) {
                    // Sort by index
                    items.sort(function(a, b) {
                        if (a[sortBy] === b[sortBy]) {
                            return 0;
                        }
                        if (a[sortBy] === null || a[sortBy] === undefined) {
                            return -1;
                        }
                        if (b[sortBy] === null || b[sortBy] === undefined) {
                            return 1;
                        }
                        if (a[sortBy].constructor === String && b[sortBy].constructor === String) {
                            return a[sortBy].localeCompare(b[sortBy], undefined, { sensivity: 'base' });
                        }
                        if (a[sortBy] instanceof Array && b[sortBy] instanceof Array) {
                            const reducedA = a[sortBy].length ? a.filament.reduce((a, b) => a + b) : 0;
                            const reducedB = b[sortBy].length ? b.filament.reduce((a, b) => a + b) : 0;
                            return reducedA - reducedB;
                        }
                        return a[sortBy] - b[sortBy];
                    });

                    // Deal with descending order
                    if (sortDesc) items.reverse();

                    // Then make sure directories come first
                    items.sort((a, b) => (a.isDirectory === b.isDirectory) ? 0 : (a.isDirectory ? -1 : 1));
                }
                return items;
            },
            loadPath: function() {
                let dirArray = this.currentPath.substring(1).split("/");

                this.files = findDirectory(this.filetree, dirArray);
                if (dirArray.length === 1 && this.currentPath === "") {
                    this.files = this.files.filter(element => element.filename !== "gcodes");
                }

                if (!this.showHiddenFiles) {
                    this.files = this.files.filter(file => file.filename.substr(0, 1) !== ".");
                }
            },
            clickRow: function(item) {
                if (!item.isDirectory) {
                    this.editor.showLoader = true;
                    this.editor.sourcecode = "";
                    this.editor.item = item;

                    let url = '//'+this.hostname+':'+this.port+'/server/files'+this.currentPath+'/'+item.filename+'?time='+Date.now();
                    fetch(url, { cache: "no-cache" }).then(res => res.text()).then(file => {
                        this.editor.sourcecode = file;
                        this.editor.show = true;
                        this.editor.showLoader = false;
                        this.editor.readonly = false;
                        if (this.currentPath === '/config_examples') this.editor.readonly = true;
                    });

                } else {
                    this.currentPath += "/"+item.filename;
                    this.currentPage = 1;
                }
            },
            clickRowGoBack: function() {
                this.currentPath = this.currentPath.substr(0, this.currentPath.lastIndexOf("/"));
            },
            saveFile: function(boolRestart = false) {
                let file = new File([this.editor.sourcecode], this.editor.item.filename);

                let formData = new FormData();
                formData.append('file', file);
                formData.append('root', 'config');
                if(this.currentPath.length > 7) formData.append('path', this.currentPath.substring(8));

                axios.post('//' + this.hostname + ':' + this.port + '/server/files/upload',
                    formData, {
                        headers: { 'Content-Type': 'multipart/form-data' }
                    }
                ).then(() => {
                    this.$toast.success("File '"+this.editor.item.filename+"' successfully saved.");
                    this.editor.show = false;
                    this.editor.sourcecode = "";

                    if (boolRestart && this.editor.item.filename === "moonraker.conf") {
                        this.$socket.sendObj('machine.services.restart', { service: "moonraker" })
                    } else if (boolRestart) {
                        this.$store.commit('server/addEvent', { message: "FIRMWARE_RESTART", type: 'command' })
                        this.$socket.sendObj('printer.gcode.script', { script: "FIRMWARE_RESTART" })
                    }
                }).catch(() => {
                    this.$toast.error("Error save "+this.editor.item.filename);
                });
            },
            showContextMenu (e, item) {
                e.preventDefault();
                this.contextMenu.shown = true;
                this.contextMenu.x = e.clientX;
                this.contextMenu.y = e.clientY;
                this.contextMenu.item = item;
                this.$nextTick(() => {
                    this.contextMenu.shown = true
                });
            },
            downloadFile() {
                let filename = (this.currentPath+"/"+this.contextMenu.item.filename);
                let link = document.createElement("a");
                link.download = name;
                link.href = '//' + this.hostname + ':' + this.port + '/server/files' + filename;
                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
            },
            createFolder() {
                this.dialogCreateFolder.name = "";
                this.dialogCreateFolder.show = true;
            },
            createFolderAction() {
                this.dialogCreateFolder.show = false;

                this.$socket.sendObj('server.files.post_directory', {
                    path: this.currentPath.substring(1)+"/"+this.dialogCreateFolder.name
                }, 'files/getCreateDir');
            },
            createFile() {
                this.dialogCreateFile.name = "";
                this.dialogCreateFile.show = true;
            },
            createFileAction() {
                let file = new File([""], this.dialogCreateFile.name);

                let formData = new FormData();
                formData.append('file', file);
                formData.append('root', 'config');
                if(this.currentPath.length > 7) formData.append('path', this.currentPath.substring(8));

                axios.post('//' + this.hostname + ':' + this.port + '/server/files/upload',
                    formData, {
                        headers: { 'Content-Type': 'multipart/form-data' }
                    }
                ).then(() => {
                    this.dialogCreateFile.show = false;
                    this.dialogCreateFile.name = "";
                }).catch(() => {
                    window.console.error("Error save "+this.editor.item.filename);
                });
            },
            renameFile(item) {
                this.dialogRenameFile.item = item;
                this.dialogRenameFile.newName = item.filename;
                this.dialogRenameFile.show = true;
            },
            renameFileAction() {
                this.dialogRenameFile.show = false;
                this.$socket.sendObj('server.files.move', {
                    source: this.currentPath+"/"+this.dialogRenameFile.item.filename,
                    dest: this.currentPath+"/"+this.dialogRenameFile.newName
                }, 'files/getMove');
            },
            removeFile() {
                this.$socket.sendObj('server.files.delete_file', { path: this.currentPath+"/"+this.contextMenu.item.filename }, 'files/getDeleteFile');
            },
            deleteDirectoryAction: function() {
                this.$socket.sendObj('server.files.delete_directory', { path: this.currentPath+"/"+this.contextMenu.item.filename }, 'files/getDeleteDir');
            },
            uploadFileButton: function() {
                this.$refs.fileUpload.click()
            },
            async uploadFile() {
                if (this.$refs.fileUpload.files.length) {
                    this.$store.commit('socket/addLoading', { name: 'configFileUpload' })
                    let successFiles = []
                    this.uploadSnackbar.number = 0
                    this.uploadSnackbar.max = this.$refs.fileUpload.files.length
                    for (const file of this.$refs.fileUpload.files) {
                        this.uploadSnackbar.number++
                        const result = await this.doUploadFile(file)
                        successFiles.push(result)
                    }

                    this.$store.commit('socket/removeLoading', { name: 'configFileUpload' })
                    for(const file of successFiles) {
                        this.$toast.success("Upload of "+file+" successful!")
                    }

                    this.$refs.fileUpload.value = ''
                }
            },
            doUploadFile: function(file) {
                let toast = this.$toast;
                let formData = new FormData();
                let filename = file.name.replace(" ", "_");

                this.uploadSnackbar.filename = filename
                this.uploadSnackbar.status = true
                this.uploadSnackbar.percent = 0
                this.uploadSnackbar.speed = 0
                this.uploadSnackbar.lastProgress.loaded = 0
                this.uploadSnackbar.lastProgress.time = 0

                formData.append('root', 'config');
                formData.append('file', file, (this.currentPath+"/"+filename).substring(7));
                this.$store.commit('socket/addLoading', { name: 'configFileUpload' });

                return new Promise(resolve => {
                    this.uploadSnackbar.cancelTokenSource = axios.CancelToken.source();
                    axios.post('//' + this.hostname + ':' + this.port + '/server/files/upload',
                        formData, {
                            headers: { 'Content-Type': 'multipart/form-data' },
                            cancelToken: this.uploadSnackbar.cancelTokenSource.token,
                            onUploadProgress: (progressEvent) => {
                                this.uploadSnackbar.percent = (progressEvent.loaded * 100) / progressEvent.total
                                if (this.uploadSnackbar.lastProgress.time) {
                                    const time = progressEvent.timeStamp - this.uploadSnackbar.lastProgress.time
                                    const data = progressEvent.loaded - this.uploadSnackbar.lastProgress.loaded

                                    if (time) this.uploadSnackbar.speed = data / (time / 1000)
                                }

                                this.uploadSnackbar.lastProgress.time = progressEvent.timeStamp
                                this.uploadSnackbar.lastProgress.loaded = progressEvent.loaded
                                this.uploadSnackbar.total = progressEvent.total
                            }
                        }
                    ).then((result) => {
                        this.uploadSnackbar.status = false
                        resolve(result.data.result)
                    }).catch(() => {
                        this.uploadSnackbar.status = false
                        this.$store.commit('socket/removeLoading', { name: 'configFileUpload' });
                        toast.error("Cannot upload the file!");
                    })
                })
            },
            cancelUpload: function() {
                this.uploadSnackbar.cancelTokenSource.cancel()
                this.uploadSnackbar.status = false
            },
        },
        watch: {
            config: {
                deep: true,
                handler() {
                    this.loadPath();
                }
            },
            config_examples: {
                deep: true,
                handler() {
                    this.loadPath();
                }
            },
            filetree: {
                deep: true,
                handler() {
                    this.loadPath();
                }
            },
            currentPath: {
                handler() {
                    this.loadPath();
                }
            },
            showHiddenFiles: {
                handler() {
                    this.loadPath();
                }
            }
        }
    }
</script>
