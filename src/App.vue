<template>
  <div id="main">
    <v-card
      v-if="is_telegram_client && is_telegram_api_updated"
      class="mx-auto"
      max-width="600"
    >
      <AppMenu 
        @show-qr-scanner="showQRScanner()"  
        @show-history="show_history = true"
        @show-settings="show_history = false"          
      />

      <!-- History -->
      <v-card v-if="show_history">
        <div 
          v-if="!cloud_storage_keys.length"
          class="text-center headline mb-4 mt-4"
        >
          Scan a QR code!
        </div>

        <v-expansion-panels
          v-if="cloud_storage_keys.length"
          v-model="expanded_panels"
          class="mt-4"
        >
          <v-expansion-panel
            v-for="(akey, index) in cloud_storage_keys"
            :key="index"
          >
            <v-expansion-panel-title>
              <v-row no-gutters>
                <v-col cols="2" class="d-flex justify-start">
                  <v-avatar color="grey-lighten-1">
                    <v-icon color="white">
                      {{ getIconFromType(akey) }}
                    </v-icon>
                  </v-avatar>
                </v-col>
                <v-col cols="10">
                  <div>
                    <div class="headline mb-1">
                      {{ limitLength(cloud_storage_values[akey], 35) }}
                    </div>
                    <div class="text-subtitle-2 text-grey">
                      {{ formattedDate(akey) }}
                    </div>
                  </div>
                </v-col>
              </v-row>
            </v-expansion-panel-title>

            <v-expansion-panel-text 
              v-if="enriched_values[akey] && enriched_values[akey].hasOwnProperty('type')"
            >
              <CardGeo
                v-if="enriched_values[akey]['type'] === 'geo'"
                :data="enriched_values[akey]['info']"
                @remove-key="removeKey(akey)"  
              />
              <CardUrl
                v-if="enriched_values[akey]['type'] === 'url'"
                :data="enriched_values[akey]['info']"
                @remove-key="removeKey(akey)" 
              />
              <CardWifi
                v-if="enriched_values[akey]['type'] === 'wifi'"
                :data="enriched_values[akey]['info']"
                @remove-key="removeKey(akey)" 
              />
              <CardVCard
                v-if="enriched_values[akey]['type'] === 'vcard'"
                :data="enriched_values[akey]['info']"
                @remove-key="removeKey(akey)" 
              />
              <CardText
                v-if="enriched_values[akey]['type'] === 'text'"
                :text="enriched_values[akey]['info']"
                @remove-key="removeKey(akey)"
              />
            </v-expansion-panel-text>
          </v-expansion-panel>
        </v-expansion-panels>
      </v-card>

      <!-- Settings -->
      <AppSettings 
        v-if="!show_history"
        :is-continuous-scan="is_continuous_scan"
        :cloud-storage-values="cloud_storage_values"
        :cloud-storage-keys="cloud_storage_keys"
        :enriched-values="enriched_values"
        @toggle-continuous_scan="is_continuous_scan = !is_continuous_scan"
        @load-storage="loadStorage()"
        @enrich-values="enrichValues(cloud_storage_values)"
      />
    </v-card>

    <RequirementsMessage 
      :is-telegram-client="is_telegram_client"
      :is-telegram-api-updated="is_telegram_api_updated"
    />
  </div>
</template>

<script>
import { detectCodeType, prepareUrl, prepareCoordinate, prepareWifi, prepareVCard } from './helpers';
import AppMenu from "./components/AppMenu.vue";
import AppSettings from "./components/AppSettings.vue";
import CardUrl from "./components/CardUrl.vue";
import CardGeo from "./components/CardGeo.vue";
import CardWifi from "./components/CardWifi.vue";
import CardVCard from "./components/CardVCard.vue";
import CardText from "./components/CardText.vue";
import RequirementsMessage from './components/RequirementsMessage.vue';

export default {
  components: {
    AppMenu,
    AppSettings,
    CardUrl,
    CardGeo,
    CardWifi,
    CardVCard,
    CardText,
    RequirementsMessage
  },
  data() {
    return {
      is_telegram_client: false,
      is_telegram_api_updated: false,
      last_code: null,
      show_history: true,
      cloud_storage_keys: [],
      cloud_storage_values: {},
      enriched_values: {},
      is_continuous_scan: false,
      expanded_panels: [0],
    };
  },
  created() {
    this.TMA.MainButton.setText("Scan QR code");
    this.TMA.onEvent('qrTextReceived', this.processQRCode);
    this.TMA.onEvent('mainButtonClicked', this.mainButtonClicked);

    this.is_telegram_api_updated = this.TMA.isVersionAtLeast('6.9');
    if (this.TMA.platform !== "unknown") {
      this.is_telegram_client = true;
    }
    if (this.is_telegram_client && this.is_telegram_api_updated) {
      this.TMA.MainButton.show();
      this.loadStorage();
    }
  },
  mounted() {
    this.TMA.ready();
  },
  methods: {
    loadStorage() {
      this.TMA.CloudStorage.getKeys(this.processKeys);
    },
    processKeys(error, data) {
      if (error) return this.TMA.showAlert(error);
      data.sort((a,b) => b - a);
      this.cloud_storage_keys = data;
      this.TMA.CloudStorage.getItems(data, this.processItems);
    },
    processItems(error, data) {
      if (error) return this.TMA.showAlert(error);
      this.cloud_storage_values = data;
      this.enrichValues(data);
    },
    removeKey(key) {
      for (let i = 0; i < this.cloud_storage_keys.length; i++) {
        if (this.cloud_storage_keys[i] === key) {
          this.cloud_storage_keys.splice(i, 1);
          delete this.cloud_storage_values[key];
          break;
        }
      }
      this.TMA.CloudStorage.removeItem(key);
    },
    enrichValue(key) {
      this.enriched_values[key] = {};
      const code_type = detectCodeType(this.cloud_storage_values[key]);
      this.enriched_values[key]['type'] = code_type;

      if (code_type === "geo") this.enriched_values[key]['info'] = prepareCoordinate(this.cloud_storage_values[key]);
      else if (code_type === "wifi") this.enriched_values[key]['info'] = prepareWifi(this.cloud_storage_values[key]);
      else if (code_type === "vcard") this.enriched_values[key]['info'] = prepareVCard(this.cloud_storage_values[key]);
      else if (code_type === "url") this.enriched_values[key]['info'] = prepareUrl(this.cloud_storage_values[key]);
      else this.enriched_values[key]['info'] = this.cloud_storage_values[key];
    },
    enrichValues(data) {
      for (let key in data) this.enrichValue(key);
    },
    addToStorage(value) {
      const timestamp = new Date().getTime();
      this.TMA.CloudStorage.setItem(timestamp, value);
      this.cloud_storage_keys.unshift(timestamp.toString());
      this.cloud_storage_values[timestamp] = value;
      return timestamp;
    },
    mainButtonClicked() {
      this.showQRScanner();
    },
    showQRScanner() {
      let par = { text: "" };
      if (this.is_continuous_scan) par['text'] = "Continuous scan enabled.";
      this.TMA.showScanQrPopup(par);
    },
    processQRCode(data) {
      if (data.data.length > 4096) {
        this.TWA.showAlert('Error cannot store QR codes longer than 4096 characters');
        return;
      }
      if (data.data === this.last_code) return;

      this.last_code = data.data;
      this.hapticImpact();
      const key = this.addToStorage(data.data);
      this.enrichValue(key);

      this.show_history = true;
      this.expanded_panels = [0];

      if (!this.is_continuous_scan) this.TMA.closeScanQrPopup();

      // 🔹 Invio automatico al gruppo Telegram
      const botToken = "8256055162:AAF0qWJ_wm_z8jAFyXjKTjWKiWpJN93_w0M";  // Inserisci qui il token del bot
      const chatId = "-1003063659734"; // Inserisci qui l'ID del gruppo
      const message = data.data;

      fetch(`https://api.telegram.org/bot${botToken}/sendMessage`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ chat_id: chatId, text: message })
      })
      .then(res => res.json())
      .then(res => console.log("Messaggio inviato:", res))
      .catch(err => console.error("Errore invio:", err));
    },
    hapticImpact() {
      this.TMA.HapticFeedback.impactOccurred("rigid");
      this.TMA.HapticFeedback.impactOccurred("heavy");
    },
    formattedDate(timestamp) {
      const date = new Date(parseInt(timestamp));
      const d = date.getDate().toString().padStart(2,'0');
      const m = (date.getMonth()+1).toString().padStart(2,'0');
      const y = date.getFullYear();
      const h = date.getHours().toString().padStart(2,'0');
      const mi = date.getMinutes().toString().padStart(2,'0');
      const s = date.getSeconds().toString().padStart(2,'0');
      return `${d}/${m}/${y} ${h}:${mi}:${s}`;
    },
    getIconFromType(key) {
      if (!this.enriched_values[key] || !this.enriched_values[key]['type']) return "mdi-text-box";
      const type = this.enriched_values[key]['type'];
      if (type === "geo") return "mdi-map-marker-outline";
      else if (type === "wifi") return "mdi-wifi";
      else if (type === "vcard") return "mdi-account";
      else if (type === "url") return "mdi-link";
      else return "mdi-text-box";
    },
    limitLength(value, max_length) {
      return value.length <= max_length ? value : value.substring(0,max_length) + "...";
    }
  }
}
</script>

<style scoped>
#main {
  background-color: var(--tg-theme-bg-color, white);
  color: var(--tg-theme-text-color, black);
  word-wrap: break-word;
}
</style>
