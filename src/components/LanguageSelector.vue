<template>
  <div class="dropdown is-hoverable">
    <div class="dropdown-trigger">
      <button class="button" aria-haspopup="true" aria-controls="dropdown-menu">
        <div class="dropdown-item">
          <img :src="'flags/gif/' + currentLocale + '.gif'"/> {{$t("changeLanguage")}}
          <span class="icon is-small">
          <i class="fas fa-angle-down" aria-hidden="true"></i>
        </span>
        </div>
      </button>
    </div>
    <div class="dropdown-menu" id="dropdown-menu" role="menu">
      <div class="dropdown-content">
        <a class="dropdown-item" v-for="locale in locales" :key="locale" @click="changeLanguage(locale)">
          <img :src="'flags/gif/' + locale + '.gif'"/> {{ $i18n.messages[locale]['languageLocalName'] }}
        </a>
        <hr class="dropdown-divider">
         <a href="https://github.com/flxn/qrcode2stl#contribute-a-translation" class="dropdown-item" rel="nofollow noopener" target="_blank">
          <i class="fab fa-github"></i> {{$t('contributeTranslation')}}
        </a>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'LanguageSelector',
  data() {
    return {
      currentLocale: this.$i18n.locale,
      locales: this.$i18n.availableLocales,
    };
  },
  methods: {
    changeLanguage(locale) {
      this.$i18n.locale = locale;
      this.currentLocale = locale;
      window.localStorage.setItem('locale', locale);
      return false;
    },
  },
};
</script>

<style>
.dropdown .dropdown-item img {
  width: 24px;
  height: 16px;
  margin-right: 8px;
}
.dropdown .dropdown-item {
  display: flex;
  align-items: center;
}
.dropdown .dropdown-item i {
  margin-right: 4px;
}
</style>
