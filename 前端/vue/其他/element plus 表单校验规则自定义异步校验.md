```javascript
const rules = {
  name: [
    {
      trigger: "blur",
      validator: async (rule, value, callback) => {
        if (
          value !== formModel.value?.name &&
          (await FavoritesApi.validateFavorites(value))
        ) {
          callback("书单名称已存在");
        }
      }
    }
  ]
};
```