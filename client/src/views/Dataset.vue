<template>
  <div>
    <div style="padding-top: 55px" />
    <div class="album py-5 bg-light" style="overflow: auto; height: calc(100vh - 55px)">
      <div class="container">
        <h2 class="text-center">Dataset of {{ dataset.name }}</h2>
        <p class="text-center">
          Total of <strong>{{ imageCount }}</strong> images displayed on <strong>{{ pages }}</strong> pages.
        </p>

        <div class="row justify-content-md-center">
          <div class="col-md-auto btn-group" role="group" style="padding-bottom: 20px">
            <button type="button" class="btn btn-success" data-toggle="modal" data-target="#generateDataset">
              Generate
            </button>
            <button type="button" class="btn btn-primary" data-toggle="modal" data-target="#cocoUpload">
              Import COCO
            </button>
            <!-- <button type="button" class="btn btn-info">
              Download COCO
            </button> -->
          </div>
        </div>
        <div v-if="subdirectories.length > 0" class="text-center">
          <h5>Subdirectories</h5>
          <button v-for="(subdirectory, subId) in subdirectories" :key="subId" class="btn badge badge-pill badge-primary category-badge" style="margin: 2px" @click="folders.push(subdirectory)">{{ subdirectory }}
          </button>
        </div>

        <ol class="breadcrumb">
          <li class="breadcrumb-item"></li>
          <li class="breadcrumb-item active">
            <button class="btn btn-sm btn-link" @click="folders = []">{{ dataset.name }} </button>
          </li>
          <li v-for="(folder, folderId) in folders" :key="folderId" class="breadcrumb-item">
            <button class="btn btn-sm btn-link" :disabled="folders[folders.length - 1] === folder" @click="removeFolder(folder)">{{ folder }}</button>
          </li>
        </ol>

        <p class="text-center" v-if="images.length < 1">No images found in directory.</p>
        <div v-else>
          <Pagination :pages="pages" @pagechange="updatePage" />
          <div class="row">
            <ImageCard v-for="image in images" :key="image.id" :image="image" />
          </div>
          <hr>
        </div>
      </div>
    </div>
    <div class="modal fade" tabindex="-1" role="dialog" id="generateDataset">
      <div class="modal-dialog" role="document">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">Generate a Dataset</h5>
            <button type="button" class="close" data-dismiss="modal" aria-label="Close">
              <span aria-hidden="true">&times;</span>
            </button>
          </div>
          <div class="modal-body">
            <form>
               <div class="form-group">
                <label>Keyword</label>
                <input class="form-control" v-model="keyword"/>
              </div>
              <div class="form-group">
                <label>Limit</label>
                <input class="form-control" type="number" v-model="generateLimit"/>
              </div>
            </form>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-primary" @click="generateDataset">Generate</button>
            <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
          </div>
        </div>
      </div>
    </div>
        <div class="modal fade" tabindex="-1" role="dialog" id="cocoUpload">
      <div class="modal-dialog" role="document">
        <div class="modal-content">
          <div class="modal-header">
            <h5 class="modal-title">Upload COCO Annotaitons</h5>
            <button type="button" class="close" data-dismiss="modal" aria-label="Close">
              <span aria-hidden="true">&times;</span>
            </button>
          </div>
          <div class="modal-body">
            <form>
              <div class="form-group">
                <label for="coco">COCO Annotation file (.json)</label>
                <input type="file" class="form-control-file" id="coco">
              </div>
            </form>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-primary" @click="uploadCoco" data-dismiss="modal">Upload</button>
            <button type="button" class="btn btn-secondary" data-dismiss="modal">Close</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";

import toastrs from "@/mixins/toastrs";

import ImageCard from "@/components/cards/ImageCard";
import Pagination from "@/components/Pagination";

import { mapMutations } from "vuex";

export default {
  name: "Dataset",
  components: { ImageCard, Pagination },
  mixins: [toastrs],
  props: {
    identifier: {
      type: [Number, String],
      required: true
    }
  },
  data() {
    return {
      pages: 1,
      generateLimit: 100,
      limit: 52,
      imageCount: 0,
      images: [],
      folders: [],
      dataset: {
        id: 0
      },
      subdirectories: [],
      status: {
        data: { state: true, message: "Loading data" }
      },
      keyword: ""
    };
  },
  methods: {
    ...mapMutations(["addProcess", "removeProcess"]),
    generateDataset() {
      if (this.keyword.length === 0) return;
      axios
        .post("/api/dataset/" + this.dataset.id + "/generate", {
          keywords: [this.keyword],
          limit: this.generateLimit
        })
        .then(() => {});
    },
    updatePage(page) {
      let process = "Loading images from dataset";
      this.addProcess(process);

      axios
        .get("/api/dataset/" + this.dataset.id + "/data", {
          params: {
            page: page,
            limit: this.limit,
            folder: this.folders.join("/")
          }
        })
        .then(response => {
          this.images = response.data.images;
          this.dataset = response.data.dataset;

          this.imageCount = response.data.pagination.total;
          this.pages = response.data.pagination.pages;

          this.subdirectories = response.data.subdirectories;

          this.removeProcess(process);
        })
        .catch(error => {
          this.axiosReqestError("Loading Dataset", error.response.data.message);
        });
    },
    removeFolder(folder) {
      let index = this.folders.indexOf(folder);
      this.folders.splice(index + 1, this.folders.length);
    },
    uploadCoco() {
      let process = "Uploading COCO annotation file";
      this.addProcess(process);

      let form = new FormData();

      let uploaded = document.getElementById("coco");
      form.append("coco", uploaded.files[0]);
      axios
        .post("/api/dataset/" + this.dataset.id + "/coco", form, {
          headers: {
            "Content-Type": "multipart/form-data"
          }
        })
        .then(() => {
          this.removeProcess(process);
        })
        .catch(error => {
          this.axiosReqestError("Loading Dataset", error.response.data.message);
          this.removeProcess(process);
        });
    }
  },
  watch: {
    folders() {
      this.updatePage();
    }
  },
  beforeRouteUpdate() {
    this.dataset.id = parseInt(this.identifier);
    this.updatePage();
  },
  created() {
    this.dataset.id = parseInt(this.identifier);
    this.updatePage();
  }
};
</script>

<style scoped>
.breadcrumb {
  padding: 0px;
  margin: 5px 0;
}

.btn-link {
  padding: 0px;
}
</style>
