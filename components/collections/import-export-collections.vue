<template>
  <modal v-if="show" @close="hideModal">
    <div slot="header">
      <ul>
        <li>
          <div class="flex-wrap">
            <h3 class="title">{{ $t("import_export") }} {{ $t("collections") }}</h3>
            <div>
              <button class="icon" @click="hideModal">
                <closeIcon class="material-icons" />
              </button>
            </div>
          </div>
          <div class="flex-wrap">
            <span
              v-tooltip="{
                content: !fb.currentUser ? $t('login_first') : $t('replace_current'),
              }"
            >
              <button :disabled="!fb.currentUser" class="icon" @click="syncCollections">
                <i class="material-icons">folder_shared</i>
                <span>{{ $t("import_from_sync") }}</span>
              </button>
            </span>
            <button
              class="icon"
              @click="openDialogChooseFileToReplaceWith"
              v-tooltip="$t('replace_current')"
            >
              <i class="material-icons">create_new_folder</i>
              <span>{{ $t("replace_json") }}</span>
              <input
                type="file"
                @change="replaceWithJSON"
                style="display: none"
                ref="inputChooseFileToReplaceWith"
                accept="application/json"
              />
            </button>
            <button
              class="icon"
              @click="openDialogChooseFileToImportFrom"
              v-tooltip="$t('preserve_current')"
            >
              <i class="material-icons">folder_special</i>
              <span>{{ $t("import_json") }}</span>
              <input
                type="file"
                @change="importFromJSON"
                style="display: none"
                ref="inputChooseFileToImportFrom"
                accept="application/json"
              />
            </button>
          </div>
        </li>
      </ul>
    </div>
    <div slot="body">
      <textarea v-model="collectionJson" rows="8"></textarea>
    </div>
    <div slot="footer">
      <div class="flex-wrap">
        <span></span>
        <span>
          <button class="icon" @click="hideModal">
            {{ $t("cancel") }}
          </button>
          <button class="icon primary" @click="exportJSON" v-tooltip="$t('download_file')">
            {{ $t("export") }}
          </button>
        </span>
      </div>
    </div>
  </modal>
</template>

<script>
import { fb } from "~/helpers/fb"
import closeIcon from "~/static/icons/close-24px.svg?inline"

export default {
  components: {
    closeIcon,
  },
  data() {
    return {
      fb,
    }
  },
  props: {
    show: Boolean,
  },
  computed: {
    collectionJson() {
      return JSON.stringify(this.$store.state.postwoman.collections, null, 2)
    },
  },
  methods: {
    hideModal() {
      this.$emit("hide-modal")
    },
    openDialogChooseFileToReplaceWith() {
      this.$refs.inputChooseFileToReplaceWith.click()
    },
    openDialogChooseFileToImportFrom() {
      this.$refs.inputChooseFileToImportFrom.click()
    },
    replaceWithJSON() {
      let reader = new FileReader()
      reader.onload = (event) => {
        let content = event.target.result
        let collections = JSON.parse(content)
        if (collections[0]) {
          let [name, folders, requests] = Object.keys(collections[0])
          if (name === "name" && folders === "folders" && requests === "requests") {
            // Do nothing
          }
        } else if (collections.info && collections.info.schema.includes("v2.1.0")) {
          collections = this.parsePostmanCollection(collections)
        } else {
          return this.failedImport()
        }
        this.$store.commit("postwoman/replaceCollections", collections)
        this.fileImported()
        this.syncToFBCollections()
      }
      reader.readAsText(this.$refs.inputChooseFileToReplaceWith.files[0])
      this.$refs.inputChooseFileToReplaceWith.value = ""
    },
    importFromJSON() {
      let reader = new FileReader()
      reader.onload = (event) => {
        let content = event.target.result
        let collections = JSON.parse(content)
        if (collections[0]) {
          let [name, folders, requests] = Object.keys(collections[0])
          if (name === "name" && folders === "folders" && requests === "requests") {
            // Do nothing
          }
        } else if (collections.info && collections.info.schema.includes("v2.1.0")) {
          //replace the variables, postman uses {{var}}, Hoppscotch uses <<var>>
          collections = JSON.parse(content.replaceAll(/{{([a-z]+)}}/gi, '<<$1>>'))
          collections.item = this.flattenPostmanFolders(collections)
          collections = this.parsePostmanCollection(collections)
        } else {
          return this.failedImport()
        }
        this.$store.commit("postwoman/importCollections", collections)
        this.fileImported()
        this.syncToFBCollections()
      }
      reader.readAsText(this.$refs.inputChooseFileToImportFrom.files[0])
      this.$refs.inputChooseFileToImportFrom.value = ""
    },
    exportJSON() {
      let text = this.collectionJson
      text = text.replace(/\n/g, "\r\n")
      let blob = new Blob([text], {
        type: "text/json",
      })
      let anchor = document.createElement("a")
      anchor.download = "hoppscotch-collection.json"
      anchor.href = window.URL.createObjectURL(blob)
      anchor.target = "_blank"
      anchor.style.display = "none"
      document.body.appendChild(anchor)
      anchor.click()
      document.body.removeChild(anchor)
      this.$toast.success(this.$t("download_started"), {
        icon: "done",
      })
    },
    syncCollections() {
      this.$store.commit("postwoman/replaceCollections", fb.currentCollections)
      this.fileImported()
    },
    syncToFBCollections() {
      if (fb.currentUser !== null) {
        if (fb.currentSettings[0].value) {
          fb.writeCollections(JSON.parse(JSON.stringify(this.$store.state.postwoman.collections)))
        }
      }
    },
    fileImported() {
      this.$toast.info(this.$t("file_imported"), {
        icon: "folder_shared",
      })
    },
    failedImport() {
      this.$toast.error(this.$t("import_failed"), {
        icon: "error",
      })
    },
    parsePostmanCollection(collection, folders = true) {
      let postwomanCollection = folders
        ? [
            {
              name: "",
              folders: [],
              requests: [],
            },
          ]
        : {
            name: "",
            requests: [],
          }
      if (folders) {
        //pick up collection name even when all children are folders
        postwomanCollection[0].name = collection.info ? collection.info.name : ""
      }
      for (let collectionItem of collection.item) {
        if (collectionItem.request) {
          if (postwomanCollection[0]) {
            postwomanCollection[0].name = collection.info ? collection.info.name : ""
            postwomanCollection[0].requests.push(this.parsePostmanRequest(collectionItem))
          } else {
            postwomanCollection.name = collection.name ? collection.name : ""
            postwomanCollection.requests.push(this.parsePostmanRequest(collectionItem))
          }
        } else if (collectionItem.item) {
          if (collectionItem.item[0]) {
            postwomanCollection[0].folders.push(this.parsePostmanCollection(collectionItem, false))
          }
        }
      }
      return postwomanCollection
    },
    parsePostmanRequest({ name, request }) {
      let pwRequest = {
        url: "",
        path: "",
        method: "",
        auth: "",
        httpUser: "",
        httpPassword: "",
        passwordFieldType: "password",
        bearerToken: "",
        headers: [],
        params: [],
        bodyParams: [],
        rawParams: "",
        rawInput: false,
        contentType: "",
        requestType: "",
        name: "",
      }

      pwRequest.name = name
      let requestObjectUrl = request.url.raw.match(/^(.+:\/\/[^\/]+|{[^\/]+})(\/[^\?]+|).*$/)
      if (requestObjectUrl) {
        pwRequest.url = requestObjectUrl[1]
        pwRequest.path = requestObjectUrl[2] ? requestObjectUrl[2] : ""
      }
      pwRequest.method = request.method
      let itemAuth = request.auth ? request.auth : ""
      let authType = itemAuth ? itemAuth.type : ""
      if (authType === "basic") {
        pwRequest.auth = "Basic Auth"
        pwRequest.httpUser =
          itemAuth.basic[0].key === "username" ? itemAuth.basic[0].value : itemAuth.basic[1].value
        pwRequest.httpPassword =
          itemAuth.basic[0].key === "password" ? itemAuth.basic[0].value : itemAuth.basic[1].value
      } else if (authType === "oauth2") {
        pwRequest.auth = "OAuth 2.0"
        pwRequest.bearerToken =
          itemAuth.oauth2[0].key === "accessToken"
            ? itemAuth.oauth2[0].value
            : itemAuth.oauth2[1].value
      } else if (authType === "bearer") {
        pwRequest.auth = "Bearer Token"
        pwRequest.bearerToken = itemAuth.bearer[0].value
      }
      let requestObjectHeaders = request.header
      if (requestObjectHeaders) {
        pwRequest.headers = requestObjectHeaders
        for (let header of pwRequest.headers) {
          delete header.name
          delete header.type
        }
      }
      let requestObjectParams = request.url.query
      if (requestObjectParams) {
        pwRequest.params = requestObjectParams
        for (let param of pwRequest.params) {
          delete param.disabled
        }
      }
      if (request.body) {
        if (request.body.mode === "urlencoded") {
          let params = request.body.urlencoded
          pwRequest.bodyParams = params ? params : []
          for (let param of pwRequest.bodyParams) {
            delete param.type
          }
        } else if (request.body.mode === "raw") {
          pwRequest.rawInput = true
          pwRequest.rawParams = request.body.raw
        }
      }
      return pwRequest
    },
    flattenPostmanFolders(collection) {
      let items = []

      for (let collectionItem of collection.item) {
        if (this.hasFolder(collectionItem)) {
          let newFolderItems = []
          for (let folderItem of collectionItem.item) {
            if (this.isSubFolder(folderItem)) {
              newFolderItems = newFolderItems.concat(this.flattenPostmanItem(folderItem))
            } else {
              newFolderItems.push(folderItem)
            }
          }
          collectionItem.item = newFolderItems
        }
        items.push(collectionItem)
      }
      return items;
    },
    hasFolder(item) {
      return item.hasOwnProperty('item')
    },
    isSubFolder(item) {
      return item.hasOwnProperty('_postman_isSubFolder') && item._postman_isSubFolder
    },
    flattenPostmanItem(subFolder, subFolderGlue = ' -- ') {
      delete subFolder._postman_isSubFolder
      let flattenedItems = []
      for (let subFolderItem of subFolder.item) {
        subFolderItem.name = subFolder.name + subFolderGlue + subFolderItem.name
        if (this.isSubFolder(subFolderItem)) {
          flattenedItems = flattenedItems.concat(this.flattenPostmanItem(subFolderItem))
        } else {
          flattenedItems.push(subFolderItem)
        }
      }
      return flattenedItems
    },
  },
}
</script>
