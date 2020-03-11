<template>
  <v-app id="pics">
    <v-main>
      <v-container fluid class="fill-height">
        <v-app-bar app dense color="green" dark>
          <h1>Pics</h1>
          <v-breadcrumbs v-if="path" :items="shownPath">
            <template v-slot:item="props">
              <v-breadcrumbs-item v-if="props.item.select">
                <v-select
                  v-if="subGalleries.length"
                  hide-details
                  hide-selected
                  single-line
                  menu-props="offset-y"
                  dense
                  v-model="subGallery"
                  :items="props.item.select"
                ></v-select>
              </v-breadcrumbs-item>
              <v-breadcrumbs-item v-else>
                <a
                  class="white--text"
                  v-if="props.item.href"
                  :href="props.item.href"
                  >{{ props.item.text }}</a
                >
                <span v-else>{{ props.item.text }}</span>
              </v-breadcrumbs-item>
            </template>
          </v-breadcrumbs>
          <v-spacer />
        </v-app-bar>
        <v-row v-if="!gallery" align="center" justify="center">
          <v-col v-if="!loading && path === ''" cols="12" sm="8" md="4">
            <v-text-field
              prepend-icon="mdi-filmstrip"
              @change="setPath"
              label="Gallery path"
            />
          </v-col>
          <v-col v-if="loading" cols="12" sm="8" md="4">
            <v-card>
              <v-card-title>Loading ..</v-card-title>
              <v-card-text>Loading {{ loading }}</v-card-text>
            </v-card>
          </v-col>
          <v-col v-if="!loading && loginRequired" cols="12" sm="8" md="4">
            <v-card>
              <v-card-title class="blue">Login</v-card-title>
              <v-card-text>
                <v-form>
                  <v-text-field
                    label="Login"
                    v-model="auth.username"
                    prepend-icon="mdi-account"
                    type="text"
                  />

                  <v-text-field
                    label="Password"
                    prepend-icon="mdi-lock"
                    type="password"
                    v-model="auth.password"
                  />
                </v-form>
                <v-alert class="error" v-if="loginFailed">Login failed</v-alert>
              </v-card-text>
              <v-card-actions>
                <v-spacer />
                <v-btn @click="loadGallery(path)">Login</v-btn>
              </v-card-actions>
            </v-card>
          </v-col>
          <v-col v-if="error" cols="12" sm="8" md="4">
            <v-card>
              <v-card-title class="error">Error</v-card-title>
              <v-card-text>{{ error }}</v-card-text>
              <v-card-actions>
                <v-spacer />
                <v-btn @click="path = ''">Dismiss</v-btn>
              </v-card-actions>
            </v-card>
          </v-col>
        </v-row>
        <v-row justify="center" v-if="gallery" style="height: 100%">
          <gallery
            :data="gallery"
            :auth="loginRequired ? auth : null"
            :opts="hashQuery"
            @load="(e) => (subGalleries = e.folders)"
          />
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>
<style>
html,
body,
.v-application,
main {
  height: 100%;
}
</style>
<script>
import Gallery from "@/components/Gallery";
import _ from "lodash";

export default {
  name: "App",

  components: { Gallery },

  data: () => ({
    path: "",
    loading: "",
    error: null,
    loginRequired: false,
    auth: {},
    loginFailed: false,
    hashQuery: null,
    subGallery: null,
    subGalleries: ["test"],
    gallery: null,
  }),
  watch: {
    path(val) {
      window.location.hash = val;
      this.error = null;
      this.gallery = null;
      this.loading = false;
      this.subGalleries = [];
      this.subGallery = "";
      if (val !== "") this.loadGallery(val);
    },
    subGallery(val) {
      this.path = this.path.replace(/\/*$/, "") + "/" + val;
    },
  },
  methods: {
    setPath(v) {
      window.location.hash = v;
    },
    loadGallery(path) {
      this.loading = `gallery from '${path}'`;
      var opts = {
        method: "PROPFIND",
        headers: { Depth: "1" },
        responseType: "document",
      };
      if (this.loginRequired) opts.auth = this.auth;
      this.$axios(path, opts)
        .then((resp) => {
          this.loading = "";
          this.gallery = resp.data;
        })
        .catch((err) => {
          this.loading = false;
          if (
            err.response &&
            err.response.status &&
            err.response.status === 401
          ) {
            if (this.loginRequired) this.loginFailed = true;
            this.loginRequired = true;
          } else this.error = err;
        });
    },
    parseHash() {
      this.path = window.location.hash.replace(/^#/, "").replace(/\?.*/, "");
      var qm = window.location.hash.match(/.*\?(.*)/);
      this.hashQuery = qm
        ? _.chain(qm[1])
            .split("&")
            .map(_.partial(_.split, _, "=", 2))
            .fromPairs()
            .value()
        : null;
    },
  },
  computed: {
    shownPath() {
      var url = new URL(this.path, window.location.href);
      let here = url.origin === document.location.origin;
      var pathArr = url.pathname.split("/").filter((p) => p);
      var ret = pathArr.map((e, i) => {
        let p = "/" + pathArr.slice(0, i + 1).join("/") + "/";
        return {
          text: e,
          href:
            i === pathArr.length - 1
              ? null
              : "#" + (here ? p : new URL(p, url).toString()),
        };
      });
      if (!here) {
        ret.unshift({ text: url.origin, href: "#" + url.origin });
      }
      if (this.subGalleries.length) {
        ret.push({
          select: this.subGalleries.map((e) => ({
            text: e.properties && e.properties.displayname,
            href: e.href,
          })),
        });
      }
      return ret;
    },
  },
  mounted() {
    window.addEventListener("hashchange", this.parseHash);
    this.parseHash();
  },
};
</script>
