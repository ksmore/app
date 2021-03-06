<template>
  <div class="login">
    <login-form class="form" />
    <transition name="slide">
      <div
        v-if="error"
        :class="errorType"
        class="notice">
        <i class="material-icons">{{ errorType }}</i>
        {{ $t(`errors[${error.code}]`) }}
        <button
          class="close"
          @click="closeError">Close error</button>
      </div>
    </transition>
    <small v-tooltip="version" class="style-4">{{ $t('powered_by_directus') }}</small>
  </div>
</template>

<script>
import { version } from "../../package.json";
import LoginForm from "../components/login-form/login-form.vue";

export default {
  name: "login",
  components: {
    LoginForm
  },
  data() {
    return {
      loading: false
    };
  },
  computed: {
    version() {
      return `${this.$t("version")} ${version}`;
    },
    errorType() {
      if (+this.error.code >= 100 && +this.error.code < 200) {
        if (+this.error.code === 101 || +this.error.code === 102) {
          return null;
        }

        return "warning";
      }
      return "error";
    },
    error() {
      return this.$store.state.auth.error;
    }
  },
  methods: {
    closeError() {
      this.$store.dispatch("removeAuthError");
    }
  }
};
</script>

<style lang="scss" scoped>
.login {
  height: 100%;
}

.form {
  width: 100%;
  max-width: 260px;
  margin: 0 auto;
  margin-top: calc(50vh - 150px); /* visually centered */
}

small {
  position: absolute;
  bottom: 20px;
  text-align: center;
  cursor: help;
  left: 0;
  right: 0;
  margin-left: auto;
  margin-right: auto;
}

.notice {
  position: absolute;
  bottom: 50px;
  background-color: var(--light-gray);
  border-radius: var(--border-radius);
  color: var(--white);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 10px 13px;
  left: 0;
  right: 0;
  margin-left: auto;
  margin-right: auto;
  width: max-content;

  i {
    margin-right: 5px;
  }
}

.notice.error {
  background-color: var(--danger-dark);
}

.notice.warning {
  background-color: var(--warning);
}

.slide-enter-active {
  transform: translateY(0);
  opacity: 1;
  transition: var(--slow) var(--transition-in);
}

.slide-leave-active {
  transform: translateY(0);
  transition: var(--medium) var(--transition-out);
}

.slide-enter,
.slide-leave-to {
  transform: translateY(calc(50px + 100%));
  opacity: 0;
}

.close {
  position: absolute;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
  z-index: 5;
  cursor: pointer;
  opacity: 0;
}
</style>
