```json
{
  "name": "public-service-management-platform",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite --host --mode dev",
    "test-build": "vite build --mode test",
    "prod-build": "vite build --mode prod",
    "preview": "vite preview --mode dev",
    "prettier": "prettier --write ./**/*.{html,vue,js,json,md}",
    "eslint": "eslint . --fix",
    "stylelint": "stylelint ./**/*.{css,less,vue,html} --fix",
    "prepare": "husky"
  },
  "dependencies": {
    "@vicons/ionicons5": "^0.13.0",
    "axios": "^1.7.9",
    "clipboard": "^2.0.11",
    "core-js": "^3.39.0",
    "crypto-js": "^4.2.0",
    "is-valid-json": "^1.0.2",
    "jsencrypt": "^3.3.2",
    "naive-ui": "^2.40.4",
    "pangju-utils": "^2.3.2",
    "spark-md5": "^3.0.2",
    "vue": "^3.5.13",
    "vue-router": "^4.5.0",
    "vue3-json-viewer": "^2.2.2"
  },
  "devDependencies": {
    "@eslint/compat": "^1.2.4",
    "@eslint/js": "^9.17.0",
    "@vitejs/plugin-vue": "^5.2.1",
    "eslint": "^9.17.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-prettier": "^5.2.1",
    "eslint-plugin-vue": "^9.32.0",
    "globals": "^15.13.0",
    "husky": "^9.1.7",
    "less": "^4.2.1",
    "lint-staged": "^15.2.11",
    "postcss": "^8.4.49",
    "postcss-html": "^1.7.0",
    "postcss-less": "^6.0.0",
    "prettier": "^3.4.2",
    "stylelint": "^16.12.0",
    "stylelint-config-recommended-less": "^3.0.1",
    "stylelint-config-standard": "^36.0.1",
    "stylelint-config-standard-vue": "^1.0.0",
    "stylelint-less": "^3.0.1",
    "stylelint-order": "^6.0.4",
    "unplugin-auto-import": "^0.18.1",
    "unplugin-vue-components": "^0.27.3",
    "vite": "^6.0.3",
    "vite-plugin-compression": "^0.5.1",
    "vite-plugin-remove-console": "^2.2.0",
    "vite-plugin-stylelint": "^6.0.0"
  },
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.{js,vue}": [
      "pnpm run eslint && pnpm run stylelint && pnpm run prettier"
    ]
  }
}

```