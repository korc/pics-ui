<template>
  <v-container fluid class="align-center" style="height: 100%">
    <v-row justify="center">
      <v-alert
        class="error"
        v-if="error"
        @input="
          (e) => {
            if (e) error = null;
          }
        "
        dismissible
        >{{ error }}</v-alert
      >
      <v-alert class="warning" v-if="items.length === 0">
        No pictures here
      </v-alert>
    </v-row>
    <v-row style="height: 100%">
      <v-img
        ref="image"
        height="100%"
        @load="imageLoaded"
        @error="(e) => (error = 'Could not load: ' + e)"
        :class="{
          rotateRight: orientation === 6,
          rotateLeft: orientation === 8,
        }"
        contain
        v-if="cur && cur.properties.contenttype.indexOf('image/') === 0"
        :src="(cur.tn && cur.tn[$vuetify.breakpoint.name]) || cur.href"
        :lazy-src="cur.tn && cur.tn['ic']"
        v-touch="{ right: prev, left: next }"
      >
        <template v-slot:placeholder>
          <v-row class="fill-height ma-0" align="center" justify="center">
            <v-progress-circular
              indeterminate
              color="blue"
            ></v-progress-circular>
          </v-row>
        </template>
        <v-container fluid class="fill-height d-flex">
          <v-row class="align-self-start">
            <v-btn icon class="px-2" @click="showImageList = true"
              ><v-icon>mdi-dots-grid</v-icon></v-btn
            >
            <v-spacer />
            <v-btn
              v-if="cur"
              :href="cur.href"
              color="#fffa"
              rounded
              elevation="2"
            >
              {{ cur.properties.displayname }}
              <span class="px-1" v-if="cur && cur.properties.contentlength">{{
                " (" + formatBytes(cur.properties.contentlength) + ")"
              }}</span>
            </v-btn>
            <v-spacer />
            <v-btn icon @click="$refs.image.$el.requestFullscreen()">
              <v-icon>mdi-fullscreen</v-icon>
            </v-btn>
          </v-row>
          <v-row>
            <v-btn :disabled="idx <= 0" @click="prev" rounded color="#fff6">
              <v-icon>mdi-chevron-left</v-icon>
            </v-btn>
            <v-spacer />
            <v-btn
              :disabled="idx >= items.length - 1"
              @click="next"
              rounded
              color="#fff6"
            >
              <v-icon>mdi-chevron-right</v-icon>
            </v-btn>
          </v-row>
          <v-row class="align-self-end">
            <v-slider
              v-if="items.length > 0"
              :max="items.length - 1"
              v-model="idx"
              ticks="always"
            >
              <template v-slot:append>
                <v-text-field
                  :value="idx + 1"
                  hide-details
                  @change="(v) => (idx = v - 1)"
                  class="mt-0 pt-0"
                  single-line
                  type="number"
                  style="width: 3em"
                ></v-text-field>
              </template>
            </v-slider>
          </v-row>
        </v-container>
      </v-img>
    </v-row>
    <v-dialog v-model="showImageList" width="85%" scrollable>
      <v-card color="#fffa">
        <v-card-title>
          Pictures ({{ items.length }})
          <v-spacer />
          <v-btn @click="showImageList = false" icon>
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        <v-card-text>
          <v-container class="d-flex flex-wrap">
            <v-img
              v-for="(img, i) in this.items"
              :key="i"
              :src="img.tn && img.tn.ic"
              width="128"
              height="128"
              :style="{
                border: '1px ' + (i == idx ? 'solid blue' : 'dashed black'),
              }"
              @click="(idx = i), (showImageList = !(img.tn && img.tn.ic))"
            >
              <v-container class="fill-height">
                <v-row justify="center" class="align-self-end">
                  <v-chip :color="i == idx ? 'primary' : null">
                    {{ img.properties && img.properties.displayname }}
                  </v-chip>
                </v-row>
              </v-container>
            </v-img>
          </v-container>
        </v-card-text>
      </v-card>
    </v-dialog>
  </v-container>
</template>
<style>
.v-image.rotateRight .v-image__image {
  transform: rotate(90deg);
}
.v-image.rotateLeft .v-image__image {
  transform: rotate(-90deg);
}
</style>
<script>
import EXIF from "exif-js";

export default {
  props: {
    data: XMLDocument,
    auth: Object,
    opts: Object,
    exifExt: { type: String, default: ".exif.json" },
    exifOrient: Boolean,
  },
  data: () => ({
    items: [],
    idx: null,
    error: null,
    orientation: null,
    showImageList: false,
  }),
  mounted() {
    this.parsePropfind(this.data);
    window.addEventListener("keyup", this.onKeyPress);
  },
  beforeDestroy() {
    window.removeEventListener("keyup", this.onKeyPress);
  },
  watch: {
    cur() {
      if ("image" in this.$refs) this.$refs.image.isLoading = true;
      this.orientation = null;
    },
  },
  computed: {
    cur() {
      if (this.items.length === 0) return null;
      return this.items[this.idx || 0];
    },
  },
  methods: {
    onKeyPress(ev) {
      if (
        ev.altKey ||
        ev.ctrlKey ||
        ev.shiftKey ||
        ev.target.tagName === "INPUT"
      )
        return;
      switch (ev.key) {
        case "ArrowRight":
          this.next();
          break;
        case "ArrowLeft":
          this.prev();
          break;
        default:
          return;
      }
      ev.preventDefault();
      ev.stopPropagation();
    },
    next() {
      if (this.idx < this.items.length - 1) this.idx++;
    },
    prev() {
      if (this.idx > 0) this.idx--;
    },
    imageLoaded(url) {
      this.error = null;
      if (url.slice(url.lastIndexOf("/")).startsWith("/.tn."))
        return (this.orientation = null);
      if (!this.exifOrient && !(this.opts && "exifOrient" in this.opts)) return;
      if (url === this.cur.href && this.cur.exif)
        return (this.orientation = this.cur.exif.Orientation);
      var loadExif = () => {
        var img = this.$refs.image.image;
        var v = this;
        EXIF.getData(img, () => {
          v.orientation = EXIF.getTag(img, "Orientation");
          if (v.cur.href === url) v.cur.exif = img.exifdata;
          if (v.exifExt && v.opts && "saveExif" in v.opts)
            v.$axios.put(url + v.exifExt, JSON.stringify(img.exifdata));
        });
      };
      if (this.exifExt && (this.cur.href !== url || this.cur.exifUrl))
        this.$axios
          .get(url + this.exifExt, { resourcetype: "json" })
          .then((result) => {
            this.orientation = result.data.Orientation;
            if (this.cur.href === url) this.cur.exif = result.data;
          })
          .catch(loadExif);
      else loadExif();
    },
    formatBytes(bytes) {
      if (bytes < 1024) return parseInt(bytes) + "b";
      if (bytes < 1024 * 1024)
        return Math.round((10 * bytes) / 1024) / 10 + "kb";
      if (bytes < 1024 * 1024 * 1024)
        return Math.round((10 * bytes) / 1024 / 1024) / 10 + "mb";
      return Math.round((10 * bytes) / 1024 / 1024 / 1024) / 10 + "gb";
    },
    parsePropfind(xml) {
      var nsr = (p) => (p === "D" ? "DAV:" : null);
      var resIter = xml.evaluate(
        "/D:multistatus/D:response",
        xml,
        nsr,
        XPathResult.ORDERED_NODE_ITERATOR_TYPE,
        null
      );
      var items = [],
        folders = [],
        thumbnails = {},
        exifs = {};
      var resNode, href, prop, propIter, propVal, properties, rtypeIter, rtype;
      for (var i = 0; (resNode = resIter.iterateNext()); i++) {
        href = new URL(
          xml.evaluate(
            "D:href",
            resNode,
            nsr,
            XPathResult.STRING_TYPE,
            null
          ).stringValue,
          xml.URL
        ).toString();
        propIter = xml.evaluate(
          "D:propstat/D:prop/*",
          resNode,
          nsr,
          XPathResult.ORDERED_NODE_ITERATOR_TYPE,
          null
        );
        properties = {};
        while ((prop = propIter.iterateNext())) {
          if (prop.namespaceURI !== "DAV:")
            // eslint-disable-next-line no-console
            console.warn(
              "Warning: property not in 'DAV:' namespace: ",
              prop.namespaceURI
            );
          switch (prop.localName) {
            case "getlastmodified":
            case "getcontentlength":
            case "getetag":
            case "getcontenttype":
            case "displayname":
              propVal = xml.evaluate(
                "text()",
                prop,
                nsr,
                XPathResult.STRING_TYPE
              ).stringValue;
              if (!propVal) break;
              if (prop.localName === "getlastmodified")
                propVal = new Date(propVal);
              properties[
                prop.localName.slice(0, 3) === "get"
                  ? prop.localName.slice(3)
                  : prop.localName
              ] = propVal;
              break;
            case "resourcetype":
              rtypeIter = xml.evaluate(
                "./*",
                prop,
                nsr,
                XPathResult.ORDERED_NODE_ITERATOR_TYPE,
                null
              );
              properties.resourcetype = {};
              while ((rtype = rtypeIter.iterateNext())) {
                properties.resourcetype[rtype.localName] = true;
              }
              break;
            case "supportedlock":
              break;
            default:
              // eslint-disable-next-line no-console
              console.warn(
                "Warning: unknown DAV property: ",
                prop.localName,
                prop
              );
              properties[prop.localName] = xml.evaluate(
                "text()",
                prop,
                nsr,
                XPathResult.STRING_TYPE
              ).stringValue;
          }
        }
        if (this.auth) {
          var u = new URL(href, window.location.href);
          u.username = this.auth.username;
          u.password = this.auth.password;
          href = u.href;
        }
        if (
          properties.displayname &&
          properties.displayname.endsWith(this.exifExt)
        )
          exifs[properties.displayname.slice(0, -this.exifExt.length)] = href;
        else if (
          "contenttype" in properties &&
          properties.displayname &&
          properties.contenttype.indexOf("image/") === 0
        ) {
          var m;
          if (!properties.displayname.startsWith("."))
            items.push({ href, properties });
          else if (
            (m = properties.displayname.match(
              /^\.tn\.(ic|xs|sm|md|lg|xl)\.(.*)/
            ))
          ) {
            if (!(m[2] in thumbnails)) thumbnails[m[2]] = {};
            thumbnails[m[2]][m[1]] = href;
          }
        } else if (
          i > 0 &&
          properties.displayname &&
          !properties.displayname.startsWith(".") &&
          "resourcetype" in properties &&
          properties.resourcetype.collection
        )
          folders.push({ href, properties });
      }
      var sortByDisplayName = (a, b) =>
        a.properties.displayname > b.properties.displayname
          ? 1
          : a.properties.displayname < b.properties.displayname
          ? -1
          : 0;
      this.items = items.sort(sortByDisplayName);
      this.items.forEach((i) => {
        i.tn = thumbnails[i.properties.displayname];
        i.exifUrl = exifs[i.properties.displayname];
      });
      this.$emit("load", {
        pics: items,
        folders: folders.sort(sortByDisplayName),
      });
    },
  },
};
</script>
