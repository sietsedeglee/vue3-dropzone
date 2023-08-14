<template>
  <div :id="id" ref="dropzoneElement" :class="{ 'vue-dropzone dropzone': includeStyling }">
    <div v-if="useCustomSlot" class="dz-message">
      <slot>Drop files here to upload</slot>
    </div>
  </div>
</template>

<script>
import {onMounted, onBeforeUnmount, ref, reactive, toRefs, computed} from 'vue';
import * as Dropzone from "dropzone";
import awsEndpoint from "../services/urlsigner";

Dropzone.autoDiscover = false;

export default {
  props: {
    id: String,
    options: Object,
    includeStyling: {
      type: Boolean,
      default: true
    },
    awss3: {
      type: Object,
      default: null
    },
    destroyDropzone: {
      type: Boolean,
      default: true
    },
    duplicateCheck: {
      type: Boolean,
      default: false
    },
    useCustomSlot: {
      type: Boolean,
      default: false
    }
  },
  setup(props, {emit}) {
    const dropzoneElement = ref(null);
    const dropzone = ref(null);

    const state = reactive({
      isS3: false,
      isS3OverridesServerPropagation: false,
      wasQueueAutoProcess: true,
    });

    const dropzoneSettings = computed(() => {
      let defaultValues = {
        thumbnailWidth: 200,
        thumbnailHeight: 200
      };
      Object.keys(props.options).forEach(function (key) {
        defaultValues[key] = props.options[key];
      });
      if (props.awss3 !== null) {
        defaultValues["autoProcessQueue"] = false;
        state.isS3 = true;
        state.isS3OverridesServerPropagation =
          props.awss3.sendFileToServer === false;
        if (props.options.autoProcessQueue !== undefined)
          state.wasQueueAutoProcess = props.options.autoProcessQueue;

        if (state.isS3OverridesServerPropagation) {
          defaultValues["url"] = files => {
            return files[0].s3Url;
          };
        }
      }
      return defaultValues;
    });

    onMounted(() => {
      dropzone.value = new Dropzone(dropzoneElement.value, dropzoneSettings.value);

      dropzone.value.on("thumbnail", (file, dataUrl) => {
        emit("vdropzone-thumbnail", file, dataUrl);
      });

      dropzone.value.on("addedfile", (file) => {
        let isDuplicate = false;
        if (props.duplicateCheck && dropzone.value.files.length) {
          for (let i = 0; i < dropzone.value.files.length - 1; i++) {
            if (
              dropzone.value.files[i].name === file.name &&
              dropzone.value.files[i].size === file.size &&
              dropzone.value.files[i].lastModifiedDate.toString() === file.lastModifiedDate.toString()
            ) {
              dropzone.value.removeFile(file);
              isDuplicate = true;
              emit("vdropzone-duplicate-file", file);
            }
          }
        }

        emit("vdropzone-file-added", file);
        if (state.isS3.value && state.wasQueueAutoProcess.value && !file.manuallyAdded) {
          getSignedAndUploadToS3(file);
        }
      });

      dropzone.value.on("addedfiles", function (files) {
        emit("vdropzone-files-added", files);
      });

      dropzone.value.on("removedfile", function (file) {
        emit("vdropzone-removed-file", file);
        if (file.manuallyAdded && dropzone.options.maxFiles !== null)
          dropzone.options.maxFiles++;
      });

      dropzone.value.on("success", function (file, response) {
        emit("vdropzone-success", file, response);
        if (state.isS3.value) {
          if (state.isS3OverridesServerPropagation.value) {
            var xmlResponse = new window.DOMParser().parseFromString(
              response,
              "text/xml"
            );
            var s3ObjectLocation = xmlResponse.firstChild.children[0].innerHTML;
            emit("vdropzone-s3-upload-success", s3ObjectLocation);
          }
          if (state.wasQueueAutoProcess.value) setOption("autoProcessQueue", false);
        }
      });

      dropzone.value.on("successmultiple", function (file, response) {
        emit("vdropzone-success-multiple", file, response);
      });

      dropzone.value.on("error", function (file, message, xhr) {
        emit("vdropzone-error", file, message, xhr);
        if (state.isS3) emit("vdropzone-s3-upload-error");
      });

      dropzone.value.on("errormultiple", function (files, message, xhr) {
        emit("vdropzone-error-multiple", files, message, xhr);
      });

      dropzone.value.on("sending", function (file, xhr, formData) {
        if (state.isS3.value) {
          if (state.isS3OverridesServerPropagation.value) {
            let signature = file.s3Signature;
            Object.keys(signature).forEach(function (key) {
              formData.append(key, signature[key]);
            });
          } else {
            formData.append("s3ObjectLocation", file.s3ObjectLocation);
          }
        }
        emit("vdropzone-sending", file, xhr, formData);
      });

      dropzone.value.on("totaluploadprogress", function (
        totaluploadprogress,
        totalBytes,
        totalBytesSent
      ) {
        emit(
          "vdropzone-total-upload-progress",
          totaluploadprogress,
          totalBytes,
          totalBytesSent
        );
      });

      dropzone.value.on("sendingmultiple", function (file, xhr, formData) {
        emit("vdropzone-sending-multiple", file, xhr, formData);
      });

      dropzone.value.on("complete", function (file) {
        emit("vdropzone-complete", file);
      });

      dropzone.value.on("completemultiple", function (files) {
        emit("vdropzone-complete-multiple", files);
      });

      dropzone.value.on("canceled", function (file) {
        emit("vdropzone-canceled", file);
      });

      dropzone.value.on("canceledmultiple", function (files) {
        emit("vdropzone-canceled-multiple", files);
      });

      dropzone.value.on("maxfilesreached", function (files) {
        emit("vdropzone-max-files-reached", files);
      });

      dropzone.value.on("maxfilesexceeded", function (file) {
        emit("vdropzone-max-files-exceeded", file);
      });

      dropzone.value.on("processing", function (file) {
        emit("vdropzone-processing", file);
      });

      dropzone.value.on("processingmultiple", function (files) {
        emit("vdropzone-processing-multiple", files);
      });

      dropzone.value.on("uploadprogress", function (file, progress, bytesSent) {
        emit("vdropzone-upload-progress", file, progress, bytesSent);
      });

      dropzone.value.on("reset", (event) => {
        emit("vdropzone-reset", event);
      });

      dropzone.value.on("queuecomplete", (event) => {
        emit("vdropzone-queue-complete", event);
      });

      dropzone.value.on("drop", (event) => {
        emit("vdropzone-drop", event);
      });

      dropzone.value.on("dragstart", (event) => {
        emit("vdropzone-drag-start", event);
      });

      dropzone.value.on("dragend", (event) => {
        emit("vdropzone-drag-end", event);
      });

      dropzone.value.on("dragenter", (event) => {
        emit("vdropzone-drag-enter", event);
      });

      dropzone.value.on("dragleave", (event) => {
        emit("vdropzone-drag-leave", event);
      });

      emit("vdropzone-mounted");
    });

    onBeforeUnmount(() => {
      if (props.destroyDropzone) {
        dropzone.value.destroy();
      }
    });

    const manuallyAddFile = (file, fileUrl) => {
      file.manuallyAdded = true;
      dropzone.value.emit("addedfile", file);
      const containsImageFileType = [".svg", ".png", ".jpg", ".jpeg", ".gif", ".webp"].some(ext => fileUrl.includes(ext));

      if (
        dropzone.value.options.createImageThumbnails &&
        containsImageFileType &&
        file.size <= dropzone.value.options.maxThumbnailFilesize * 1024 * 1024
      ) {
        if (fileUrl) dropzone.value.emit("thumbnail", file, fileUrl);

        const thumbnails = file.previewElement.querySelectorAll("[data-dz-thumbnail]");
        thumbnails.forEach(thumbnail => {
          thumbnail.style.width = dropzoneSettings.value.thumbnailWidth + "px";
          thumbnail.style.height = dropzoneSettings.value.thumbnailHeight + "px";
          thumbnail.style["object-fit"] = "contain";
        });
      }
      dropzone.value.emit("complete", file);
      if (dropzone.value.options.maxFiles) dropzone.value.options.maxFiles--;
      dropzone.value.files.push(file);
      emit("vdropzone-file-added-manually", file);
    };

    const setOption = (option, value) => {
      dropzone.value.options[option] = value;
    };

    const removeAllFiles = (bool) => {
      dropzone.value.removeAllFiles(bool);
    };

    const processQueue = () => {
      if (state.isS3.value && !state.wasQueueAutoProcess.value) {
        getQueuedFiles().forEach(file => {
          getSignedAndUploadToS3(file);
        });
      } else {
        dropzone.value.processQueue();
      }

      dropzone.value.on("success", function () {
        dropzone.value.options.autoProcessQueue = true;
      });

      dropzone.value.on("queuecomplete", function () {
        dropzone.value.options.autoProcessQueue = false;
      });
    };

    const init = () => dropzone.value.init();
    const destroy = () => dropzone.value.destroy();
    const updateTotalUploadProgress = () => dropzone.value.updateTotalUploadProgress();
    const getFallbackForm = () => dropzone.value.getFallbackForm();
    const getExistingFallback = () => dropzone.value.getExistingFallback();
    const setupEventListeners = () => dropzone.value.setupEventListeners();
    const removeEventListeners = () => dropzone.value.removeEventListeners();
    const disable = () => dropzone.value.disable();
    const enable = () => dropzone.value.enable();
    const filesize = (size) => dropzone.value.filesize(size);
    const accept = (file, done) => dropzone.value.accept(file, done);
    const addFile = (file) => dropzone.value.addFile(file);
    const removeFile = (file) => dropzone.value.removeFile(file);
    const getAcceptedFiles = () => dropzone.value.getAcceptedFiles();
    const getRejectedFiles = () => dropzone.value.getRejectedFiles();
    const getFilesWithStatus = () => dropzone.value.getFilesWithStatus();
    const getQueuedFiles = () => dropzone.value.getQueuedFiles();
    const getUploadingFiles = () => dropzone.value.getUploadingFiles();
    const getAddedFiles = () => dropzone.value.getAddedFiles();
    const getActiveFiles = () => dropzone.value.getActiveFiles();

    const getSignedAndUploadToS3 = (file) => {
      const promise = awsEndpoint.sendFile(
        file,
        state.awss3.value,
        state.isS3OverridesServerPropagation.value
      );

      if (!state.isS3OverridesServerPropagation.value) {
        promise.then(response => {
          if (response.success) {
            file.s3ObjectLocation = response.message;
            setTimeout(() => dropzone.value.processFile(file));
            emit("vdropzone-s3-upload-success", response.message);
          } else {
            const errorMessage = response.message || "Network Error : Could not send request to AWS. (Maybe CORS error)";
            emit("vdropzone-s3-upload-error", errorMessage);
          }
        });
      } else {
        promise.then(() => {
          setTimeout(() => dropzone.value.processFile(file));
        });
      }

      promise.catch(error => {
        alert(error);
      });
    };

    const setAWSSigningURL = (location) => {
      if (state.isS3.value) {
        state.awss3.value.signingURL = location;
      }
    };

    return {
      ...toRefs(state),
      dropzoneElement,
      dropzone,
      init,
      destroy,
      updateTotalUploadProgress,
      getFallbackForm,
      getExistingFallback,
      setupEventListeners,
      removeEventListeners,
      disable,
      enable,
      filesize,
      accept,
      addFile,
      removeFile,
      getAcceptedFiles,
      getRejectedFiles,
      getFilesWithStatus,
      getQueuedFiles,
      getUploadingFiles,
      getAddedFiles,
      getActiveFiles
      manuallyAddFile,
      setOption,
      removeAllFiles,
      processQueue,
      getSignedAndUploadToS3,
      setAWSSigningURL,
    };
  },
};
</script>

<style>
.vue-dropzone {
  border: 2px solid #e5e5e5;
  font-family: "Arial", sans-serif;
  letter-spacing: 0.2px;
  color: #777;
  transition: 0.2s linear;
}

.vue-dropzone:hover {
  background-color: #f6f6f6;
}

.vue-dropzone > i {
  color: #ccc;
}

.vue-dropzone > .dz-preview .dz-image {
  border-radius: 0;
  width: 100%;
  height: 100%;
}

.vue-dropzone > .dz-preview .dz-image img:not([src]) {
  width: 200px;
  height: 200px;
}

.vue-dropzone > .dz-preview .dz-image:hover img {
  transform: none;
  -webkit-filter: none;
}

.vue-dropzone > .dz-preview .dz-details {
  bottom: 0;
  top: 0;
  color: white;
  background-color: rgba(33, 150, 243, 0.8);
  transition: opacity 0.2s linear;
  text-align: left;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename {
  overflow: hidden;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename span,
.vue-dropzone > .dz-preview .dz-details .dz-size span {
  background-color: transparent;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename:not(:hover) span {
  border: none;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename:hover span {
  background-color: transparent;
  border: none;
}

.vue-dropzone > .dz-preview .dz-progress .dz-upload {
  background: #cccccc;
}

.vue-dropzone > .dz-preview .dz-remove {
  position: absolute;
  z-index: 30;
  color: white;
  margin-left: 15px;
  padding: 10px;
  top: inherit;
  bottom: 15px;
  border: 2px white solid;
  text-decoration: none;
  text-transform: uppercase;
  font-size: 0.8rem;
  font-weight: 800;
  letter-spacing: 1.1px;
  opacity: 0;
}

.vue-dropzone > .dz-preview:hover .dz-remove {
  opacity: 1;
}

.vue-dropzone > .dz-preview .dz-success-mark,
.vue-dropzone > .dz-preview .dz-error-mark {
  margin-left: auto;
  margin-top: auto;
  width: 100%;
  top: 35%;
  left: 0;
}

.vue-dropzone > .dz-preview .dz-success-mark svg,
.vue-dropzone > .dz-preview .dz-error-mark svg {
  margin-left: auto;
  margin-right: auto;
}

.vue-dropzone > .dz-preview .dz-error-message {
  margin-left: auto;
  margin-right: auto;
  left: 0;
  width: 100%;
  text-align: center;
}

.vue-dropzone > .dz-preview .dz-error-message:after {
  display: none;
}
</style>
