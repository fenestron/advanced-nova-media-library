<template>
  <div class="gallery" :class="{editable}">
    <cropper v-if="field.type === 'media' && editable" :image="cropImage" @close="cropImage = null" @crop-completed="onCroppedImage" :configs="field.croppingConfigs"/>

    <component :is="draggable ? 'draggable' : 'div'" v-if="images.length > 0" v-model="images"
               class="gallery-list clearfix">

      <component :is="singleComponent" v-for="(image, index) in images" class="mb-3 p-3 mr-3"
                    :key="index" :image="image" :field="field" :editable="editable" :removable="editable" @remove="remove(index)"
                    :is-custom-properties-editable="customProperties && customPropertiesFields.length > 0"
                    @edit-custom-properties="customPropertiesImageIndex = index"
                    @crop-start="cropImage = $event"
                    />

      <CustomProperties
        v-if="customPropertiesImageIndex !== null"
        v-model="images[customPropertiesImageIndex]"
        :fields="customPropertiesFields"
        @close="customPropertiesImageIndex = null"
      />

    </component>

    <span v-else-if="!editable" class="mr-3">&mdash;</span>

    <span v-if="editable" class="form-file">
      <input :id="`__media__${field.attribute}`" ref="file" class="form-file-input" type="file" @change="add"/>
      <label v-if="this.images.length < this.field.limit" :for="`__media__${field.attribute}`" class="form-file-btn btn btn-default btn-primary" v-text="label"/>
    </span>

    <p v-if="hasError" class="my-2 text-danger">
      {{ firstError }}
    </p>
    <p v-if="this.realError" class="my-2 text-danger">
        {{ this.realError }}
    </p>
  </div>
</template>

<script>
  import SingleMedia from './SingleMedia';
  import SingleFile from './SingleFile';
  import Cropper from './Cropper';
  import CustomProperties from './CustomProperties';
  import Draggable from 'vuedraggable';

  export default {
    components: {
      Draggable,
      SingleMedia,
      SingleFile,
      CustomProperties,
      Cropper,
    },
    props: {
      hasError: Boolean,
      firstError: String,
      field: Object,
      value: Array,
      editable: Boolean,
      multiple: Boolean,
      customProperties: {
        type: Boolean,
        default: false,
      },
    },
    data() {
      return {
        cropImage: null,
        images: this.value,
        customPropertiesImageIndex: null,
        singleComponent: this.field.type === 'media' ? SingleMedia : SingleFile,
        realError: "",
      };
    },
    computed: {
      draggable() {
        return this.editable && this.multiple;
      },
      customPropertiesFields() {
        return this.field.customPropertiesFields || [];
      },
      label() {
        const type = this.field.type === 'media' ? 'Media' : 'File';

        if (this.multiple || this.images.length === 0) {
          return this.__(`Add New ${type}`);
        }

        return this.__(`Replace ${type}`);
      }
    },
    watch: {
      images() {
        this.$emit('input', this.images);
      },
      value(value) {
        this.images = value;
      },
    },
    methods: {
      remove(index) {
        this.images = this.images.filter((value, i) => i !== index);
      },

      onCroppedImage(image) {
        let index = this.images.indexOf(this.cropImage);
        this.images[index] = Object.assign(image, { custom_properties: this.cropImage.custom_properties });
      },

      add() {
        if (this.images.length + 1 > this.field.limit) {
          return;
        }

        let file = this.$refs.file.files[0];

        if (file.size > 6291456) {
          this.realError = "Размер файла не должен превышать 6Мб";
          return;
        }
        if (['image/jpeg', 'image/pjpeg', 'image/png', 'application/pdf', 'application/msword',
          'application/vnd.openxmlformats-officedocument.wordprocessingml.document'].indexOf(file.type) === -1)
        {
            this.realError = "Файл должен быть типа png, jpg или jpeg";
            return;
        }

        file = new File([file], file.name, {type: file.type});
        let reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => {
          const fileData = {
            file: file,
            __media_urls__: {
              __original__: reader.result,
              default: reader.result,
            },
            name: file.name,
            file_name: file.name,
          };

          if (this.multiple) {
            this.images.push(fileData);
          } else {
            this.images = [fileData];
          }
        };

        this.realError = "";

        // reset file input so if you upload the same image sequentially
        this.$refs.file.value = null;
      },
    },
  };
</script>

<style lang="scss">
  .gallery {
    &.editable {
      .gallery-item {
        cursor: grab;
      }
    }
  }
</style>
