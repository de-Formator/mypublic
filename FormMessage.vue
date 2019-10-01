<template>
  <div class="message -form">
    <div class="content">
      <div v-if="text" class="text">
        {{ text }}
      </div>

      <form
        class="form -white"
        :class="{ '-sent': message.isSent() }"
        @submit.prevent="sendForm"
      >
        <div class="item -fields">
          <template v-for="field in activeFields">
            <div :key="field.key" class="field-wrap">
              <div v-if="$v.form[field.key].$error" class="error-text">
                {{ validationError(field.key) }}
              </div>

              <input
                v-if="isTextInput(field)"
                v-model.trim="$v.form[field.key].$model"
                :type="field.type"
                :name="field.key"
                :maxlength="field.maxlength"
                :required="field.required"
                :placeholder="fieldTitle(field)"
                :class="{ error: $v.form[field.key].$error }"
                class="form-field -input"
                @focus="hideHeaderChat"
                @blur="showHeaderChat"
              />

              <textarea
                v-if="isTextarea(field)"
                v-model.trim="$v.form[field.key].$model"
                :type="field.type"
                :name="field.key"
                :maxlength="field.maxlength"
                :required="field.required"
                :placeholder="fieldTitle(field)"
                :class="{ error: $v.form[field.key].$error }"
                class="form-field -textarea"
                @focus="hideHeaderChat"
                @blur="showHeaderChat"
              />
            </div>
          </template>

          <div
            v-if="shouldAgree && message.isNew()"
            class="field-wrap -transparent"
          >
            <label class="checkbox-wrap">
              <input v-model="agree" type="checkbox" name="checkbox" />

              <span class="fake-input">
                <SvgIcon icon="check" classes="checkmark" />
              </span>

              <!-- eslint-disable-next-line vue/no-v-html -->
              <span class="title" v-html="privacyText" />
            </label>
          </div>
        </div>

        <div>
          <div class="item -button">
            <button
              class="button accent-color-bg submit-form-button"
              :disabled="!allowSubmit"
            >
              <template v-if="!message.isSent()">
                <SvgIcon :icon="buttonIcon" classes="item -icon" />
                <span v-t="'FORM.SUBMIT_BTN'" class="item -title"></span>
              </template>

              <span v-else v-t="'FORM.SUBMITTED'" class="item -title"></span>
            </button>
          </div>
        </div>
      </form>
    </div>
  </div>
</template>

<script>
import { email, requiredIf, maxLength } from 'vuelidate/lib/validators';
import Message from '@/models/Message';
import SvgIcon from '@/components/SvgIcon.vue';
import phone from '@/validators/phone';
import headerChat from '@/services/chat/header';
import emptyValuesObject from '@/utils/emptyValuesObject';

export default {
  components: {
    SvgIcon,
  },

  props: {
    message: {
      type: Message,
      required: true,
    },

    text: {
      type: String,
      default: null,
      required: false,
    },

    fields: {
      type: Array,
      required: true,
    },

    values: {
      type: Object,
      default: function() {
        return {};
      },
      required: false,
    },

    buttonIcon: {
      type: String,
      default: 'plane',
      required: false,
    },

    privacy: {
      type: Object,
      default: null,
      required: false,
    },
  },

  data() {
    const data = {
      options: {
        fields: {},
      },
      agree: false,
      form: {},
    };

    this.fields.forEach(field => {
      if (!field.hide) {
        data.form[field.key] = this.values[field.key] || '';
        data.options.fields[field.key] = field;
      }
    });

    return data;
  },

  computed: {
    activeFields() {
      return this.fields.filter(item => !item.hide);
    },

    allowSubmit() {
      if (this.shouldAgree) {
        return this.agree;
      }

      return true;
    },

    shouldAgree() {
      return this.privacy && this.privacy.active && this.privacy.text;
    },

    privacyText() {
      return this.privacy.text;
    },
  },

  watch: {
    form: {
      // if change any field
      handler() {
        this.message.setIsNew();
      },

      deep: true,
    },
  },

  validations() {
    const data = { form: {} };

    this.fields.forEach(field => {
      if (!field.hide) {
        data.form[field.key] = {
          required: requiredIf(() => field.required),
          maxLength: maxLength(field.maxlength),
        };

        if (field.type === 'email') {
          data.form[field.key] = Object.assign(data.form[field.key], { email });
        }

        if (field.type === 'tel') {
          data.form[field.key] = Object.assign(data.form[field.key], { phone });
        }
      }
    });

    return data;
  },

  methods: {
    hideHeaderChat() {
      headerChat.hide();
    },

    showHeaderChat() {
      headerChat.show();
    },

    isTextInput(field) {
      const types = ['text', 'email', 'tel'];
      return types.indexOf(field.type) !== -1;
    },

    isTextarea(field) {
      return field.type === 'textarea';
    },

    isCheckbox(field) {
      return field.type === 'checkbox';
    },

    fieldTitle(field) {
      if (field.title.length) {
        return field.title;
      }

      return this.$t(`FORM.${field.titleKey}`);
    },

    validationError(fieldKey) {
      const options = this.options.fields[fieldKey];
      const message = [];

      switch (options.type) {
        case 'email':
          message.push(this.$t('FORM.INVALID_EMAIL_ERROR'));
          break;
        case 'tel':
          message.push(this.$t('FORM.INVALID_TEL_ERROR'));
          break;
      }

      if (options.maxLength) {
        message.push(
          this.$t('FORM.MAX_LENGTH_ERROR', { length: options.maxlength })
        );
      }

      if (options.required) {
        message.push(this.$t('FORM.REQUIRED_ERROR'));
      }

      return message.join('. ');
    },

    sendForm() {
      if (
        this.$v.$invalid ||
        emptyValuesObject(this.form) ||
        this.message.isSent()
      ) {
        return;
      }

      this.$emit('submit', {
        message: this.message,
        form: Object.assign({}, this.form),
        component: this,
      });

      this.$v.$reset();
    },
  },
};
</script>

<style>
.form {
  display: block;
}

.form.-white {
  background-color: #fff;
  border-radius: 5px;
  box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.1);
}

.form > .item {
  margin-bottom: 15px;
}

.form.-white > .item {
  margin-left: 15px;
  margin-right: 15px;
}

.form.-white > .item,
.form > .item:last-child {
  margin-bottom: 0;
}

.field-wrap {
  background-color: #fff;
  overflow: hidden;
  border-bottom: 1px solid #eee;
  transition: border-bottom 0.15s linear;
}

.form.-white .field-wrap:focus-within {
  border-bottom-color: #ccc;
}

.field-wrap:first-child {
  border-top-left-radius: 5px;
  border-top-right-radius: 5px;
}

.field-wrap:last-child {
  border-bottom-left-radius: 5px;
  border-bottom-right-radius: 5px;
  border-bottom: 0;
}

.field-wrap.-transparent {
  background-color: transparent;
  border-radius: 0;
  border-bottom: 0;
}

.form-field {
  display: block;
  font-family: sans-serif;
  font-size: 16px;
  color: #222;
  padding: 12px 15px;
  width: 100%;
  border: 0;
  box-sizing: border-box;
  background-color: transparent;
}

.form.-white .form-field {
  padding-left: 0;
  padding-right: 0;
}

.field-wrap.-wobutton:last-child {
  margin-bottom: 5px;
}

.form-field.-textarea {
  height: 80px;
}

.button {
  color: #fff;
  font-family: sans-serif;
  font-size: 14px;
  box-sizing: border-box;
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
  background-color: #495056;
  border-radius: 5px;
  border: 0;
  cursor: pointer;
  transition: opacity 0.15s linear;
  width: 100%;
  min-height: 44px;
  height: 18px;
  margin-bottom: 5px;
}

.form.-white .button {
  border-top-left-radius: 0;
  border-top-right-radius: 0;
}

.button.disabled,
.button[disabled] {
  background-color: #ccc !important;
  color: #222 !important;
}

.form.-sent .button {
  background-color: rgba(0, 0, 0, 0.05) !important;
  color: #222 !important;
}

.app.-light .button {
  color: #222;
}

.app.-cursor .button:hover {
  opacity: 0.85;
}

.button:focus,
.app.-cursor .button:focus {
  opacity: 1;
}

.button > .item {
  margin-right: 10px;
  margin-bottom: 0;
}

.button > .item:last-child {
  margin-right: 0;
}

.app.-rtl .button > .item {
  margin-right: 0;
  margin-left: 10px;
}

.app.-rtl .button > .item:last-child {
  margin-left: 0;
}

.button > .item.-icon {
  width: 18px;
  height: 18px;
}

.checkbox-wrap {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;
  align-items: flex-start;
  user-select: none;
  padding: 15px 0;
}

.checkbox-wrap input[type='checkbox'] {
  opacity: 0;
  position: absolute;
}

.fake-input {
  width: 18px;
  min-width: 18px;
  height: 18px;
  min-height: 18px;
  border-radius: 3px;
  background-color: #fff;
  margin-right: 10px;
  padding: 4px;
  box-sizing: border-box;
  color: #222;
  border: 1px solid #ccc;
}

.checkmark {
  transition: all 0.15s linear;
  opacity: 0;
  transform: scale(0.5);
}

input:checked + .fake-input .checkmark {
  opacity: 1;
  transform: scale(1);
}

.error-text {
  padding: 12px 15px 0 15px;
  color: rgba(244, 67, 54, 0.75);
  font-size: 12px;
}

.form.-white .error-text {
  padding-left: 0;
}
</style>

<style scoped>
.text {
  margin-bottom: 15px;
  text-align: center;
  color: #999;
}

.text:last-child {
  margin-bottom: 0;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.15s linear, color 0.15s linear;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>
