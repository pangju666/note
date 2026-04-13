```javascript
import axios from "axios";
import { StringUtils } from "pangju-utils";
import { createDiscreteApi, lightTheme } from "naive-ui";
import { getToken, redirectToLogin } from "@/utils/token.js";

const binaryTypes = ["blob", "arraybuffer", "stream"];

const { notification } = createDiscreteApi(["notification"], {
  configProviderProps: {
    theme: lightTheme,
  },
});

// 创建请求实例
const apiAxios = axios.create({
  // 请求超时时间设置
  timeout: 60 * 1000,
  crossDomain: true,
});

apiAxios.interceptors.request.use(
  (requestConfig) => {
    const reqConfig = { ...requestConfig };
    /* reqConfig.headers.Authorization =
      "Bearer " + import.meta.env.VITE_ACCESS_TOKEN;*/

    if (!reqConfig.url) {
      throw new Error({
        source: "axiosInterceptors",
        message: "request need url",
      });
    }
    if (reqConfig.method === "post") {
      let hasFile = false;
      for (let key of Object.keys(reqConfig.data)) {
        if (typeof reqConfig.data[key] === "object") {
          const item = reqConfig.data[key];
          if (
            item instanceof FileList ||
            item instanceof File ||
            item instanceof Blob
          ) {
            hasFile = true;
            break;
          }
        }
      }
      if (hasFile) {
        const formData = new FormData();
        Object.keys(reqConfig.data).forEach((key) => {
          if (ObjectUtils.nonNull(reqConfig.data[key])) {
            formData.append(key, reqConfig.data[key]);
          }
        });
        reqConfig.data = formData;
      }
    }
    return reqConfig;
  },
  (error) => {
    Promise.reject(error);
  },
);

apiAxios.interceptors.response.use(
  (res) => {
    if (["blob", "arraybuffer", "stream"].includes(res.config.responseType)) {
      return res;
    }

    const resData = res.data;
    if (resData.code < 0) {
      notification.error({ content: resData.message });
      return false;
    } else if (ObjectUtils.isNull(resData.data)) {
      return true;
    }
    return resData.data;
  },
  (error) => {
    if (error.code === "ECONNABORTED" && error.message.includes("timeout")) {
      notification.error({ content: "请求超时, 请稍后重试" });
      return;
    }
    notification.error({ content: error.response.data.message });
    return Promise.reject(error.response);
  },
);

export function get(url, params = {}, headers = {}, callback = () => {}) {
  try {
    return apiAxios.get(url, { params, headers });
  } catch (error) {
    callback(error);
  }
}

export function post(
  url,
  data = {},
  params = {},
  headers = {},
  callback = () => {},
) {
  try {
    return apiAxios.post(url, data, { params, headers });
  } catch (error) {
    callback(error);
  }
}

export function timeoutPost(
  url,
  data = {},
  params = {},
  headers = {},
  timeout = 60 * 1000 * 10,
  callback = () => {},
) {
  try {
    return apiAxios.post(url, data, { params, headers, timeout });
  } catch (error) {
    callback(error);
  }
}

export function del(
  url,
  data = {},
  params = {},
  headers = {},
  callback = () => {},
) {
  try {
    return apiAxios.delete(url, { data, params, headers });
  } catch (error) {
    callback(error);
  }
}

export function put(
  url,
  data = {},
  params = {},
  headers = {},
  callback = () => {},
) {
  try {
    return apiAxios.put(url, data, { params, headers });
  } catch (error) {
    callback(error);
  }
}

const fileApiAxiosConfig = {
  // 请求超时时间设置
  timeout: 1000 * 60 * 60,
  crossDomain: true,
};
const fileAxios = axios.create(fileApiAxiosConfig);

export function getDownload(
  url,
  params = {},
  headers = {},
  onDownloadProgress,
) {
  return fileAxios.get(url, {
    params,
    headers,
    responseType: "blob",
    onDownloadProgress,
  });
}

export function postDownload(
  url,
  data = {},
  params = {},
  headers = {},
  onDownloadProgress,
) {
  return fileAxios.post(url, data, {
    responseType: "blob",
    params: params,
    headers,
    onDownloadProgress,
  });
}

```