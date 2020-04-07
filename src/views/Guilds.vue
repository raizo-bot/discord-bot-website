<template>
  <v-row>
    <v-col cols="3">
      <v-card outlined>
        <v-card-title>Server</v-card-title>
        <v-card-text>
          <v-row dense>
            <v-col v-if="!guilds || guilds.length === 0"
              >Kein Server geladen oder vorhanden</v-col
            >
            <v-col v-else cols="12">
              <v-card
                v-for="guild in guilds"
                :key="guild.id"
                :class="{ 'mx-3': guild !== activeGuild, 'my-3': true }"
                :elevation="guild === activeGuild ? 5 : 0"
                @click="
                  () => {
                    activeGuild = guild;
                  }
                "
                :color="getPalette(guild.id).first"
              >
                <div class="d-flex flex-no-wrap justify-space-around">
                  <div :style="{ color: getPalette(guild.id).second }">
                    <v-card-title class="headline">
                      {{ guild.name }}
                    </v-card-title>
                    <v-card-subtitle
                      :style="{ color: getPalette(guild.id).second }"
                    >
                      <div>{{ guild.sounds.length }} Sounds verfügbar</div>
                      <div class="body-2 font-weight-thin">
                        <span>Kommando-Symbol:</span>
                        <span class="font-weight-bold">{{
                          guild.commandPrefix
                        }}</span>
                      </div>
                    </v-card-subtitle>
                  </div>
                  <v-avatar
                    class="ma-3"
                    :size="guild !== activeGuild ? '75' : '85'"
                  >
                    <v-img
                      v-if="guild.icon"
                      :ref="`img-${guild.id}`"
                      :src="guild.icon"
                    ></v-img>
                  </v-avatar>
                </div>
              </v-card>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col v-if="activeGuild" cols="9">
      <v-card outlined>
        <v-card-title>
          <span>
            <v-avatar
              :color="!activeGuild.icon ? 'primary' : 'none'"
              class="mr-3"
              size="50"
            >
              <v-img v-if="activeGuild.icon" :src="activeGuild.icon"></v-img>
              <span style="color: white" v-else>
                {{ activeGuild.name.toUpperCase().charAt(0) }}
              </span>
            </v-avatar>
          </span>
          <span v-if="!!activeGuild" class="display-1">{{
            activeGuild.name
          }}</span>
          <v-spacer></v-spacer>

          <v-dialog v-model="addSoundDialog" persistent max-width="600px">
            <template v-slot:activator="{ on }">
              <v-btn color="primary" dark v-on="on">
                Sound Hinzufügen
                <v-icon>mdi-plus</v-icon>
              </v-btn>
            </template>
            <v-card>
              <v-form
                ref="addSoundForm"
                @submit.prevent="submitUploadSoundForm"
              >
                <v-card-title>
                  <span class="headline">Neuen Sound hinzufügen</span>
                </v-card-title>
                <v-card-text>
                  <v-text-field
                    counter
                    class="mb-5"
                    :rules="validationRules.command"
                    placeholder="Befehl"
                    v-model="addSoundFormData.command"
                    :hint="
                      `Befehl ohne ${activeGuild.commandPrefix} am Anfang eingeben`
                    "
                    required
                  >
                    <template v-slot:prepend>
                      <span class="title">{{ activeGuild.commandPrefix }}</span>
                    </template>
                  </v-text-field>
                  <v-textarea
                    counter
                    :rules="validationRules.description"
                    filled
                    v-model="addSoundFormData.description"
                    placeholder="Beschreibung für den Sound"
                    required
                  ></v-textarea>
                  <v-file-input
                    placeholder="Datei auswählen"
                    show-size
                    v-model="addSoundFormData.file"
                    accept="audio/flac, audio/mp3"
                    required
                  ></v-file-input>
                </v-card-text>
                <v-card-actions>
                  <v-spacer></v-spacer>
                  <v-btn
                    color="blue darken-1"
                    text
                    @click="closeUploadSoundDialog"
                    >Close</v-btn
                  >
                  <v-btn color="blue darken-1" text type="submit"
                    >Speichern</v-btn
                  >
                </v-card-actions>
              </v-form>
            </v-card>
          </v-dialog>
          <v-spacer></v-spacer>
          <v-menu :close-on-content-click="false">
            <template v-slot:activator="{ on }">
              <v-btn color="primary" dark v-on="on" icon>
                <v-icon>mdi-sort-variant</v-icon>
              </v-btn>
            </template>
            <v-card>
              <v-card-title>Sortierung</v-card-title>
              <v-card-text>
                <v-btn-toggle v-model="sortDirection">
                  <v-btn :value="-1" icon>
                    <v-icon>mdi-sort-descending</v-icon>
                  </v-btn>
                  <v-btn :value="1" icon>
                    <v-icon>mdi-sort-ascending</v-icon>
                  </v-btn>
                </v-btn-toggle>
                <v-radio-group v-model="sortMethod">
                  <v-radio
                    v-for="(item, i) in sortings"
                    :key="item.name"
                    :label="item.name"
                    :value="i"
                  ></v-radio>
                </v-radio-group>
              </v-card-text>
            </v-card>
          </v-menu>
          <v-btn
            :loading="fetchingGuilds"
            large
            @click="reload"
            class="mr-5"
            icon
          >
            <v-icon>mdi-refresh</v-icon>
          </v-btn>
          <v-text-field
            class="mx-4"
            v-model="soundSearchString"
            flat
            hide-details
            label="Suchen"
            clearable
            prepend-inner-icon="mdi-magnify"
            solo-inverted
          ></v-text-field>
        </v-card-title>
        <v-card-text>
          <v-row v-if="activeGuild && activeGuild.sounds.length > 0">
            <v-col
              v-for="(sound, i) in getPaginatedSounds"
              :key="sound.id + i"
              cols="3"
            >
              <sound-list-tile
                @joinValueChanged="soundJoinValueChanged(sound, $event)"
                :commandPrefix="activeGuild.commandPrefix"
                :sound="sound"
                :guildId="activeGuild.id"
                :editable="sound.creator || activeGuild.owner"
                :isJoinSound="activeGuild.joinSound === sound.id"
              ></sound-list-tile>
            </v-col>
          </v-row>
          <v-row v-else>
            <v-col>Bisher keine Sounds verfügbar</v-col>
          </v-row>
          <v-row v-if="paginationLength > 1">
            <v-col>
              <v-pagination
                :length="paginationLength"
                v-model="currentSoundPage"
                :page="currentSoundPage"
                total-visible="10"
              ></v-pagination>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col v-else cols="9"></v-col>
  </v-row>
</template>
<script>
import { mapGetters, mapActions } from "vuex";
import getColors from "get-image-colors";
import chroma from "chroma-js";
import axios from "axios";

import SoundListTile from "../components/SoundListTile";

export default {
  components: {
    "sound-list-tile": SoundListTile
  },
  created() {
    console.log("guilds", this.guilds.length);
    if (!this.guilds || this.guilds.length === 0) {
      console.log("fetching guilds");
      this.reload();
    }
  },
  watch: {
    $route: {
      immediate: true,
      handler(to) {
        this.activeGuild = { id: to.query.guild };
      }
    },
    guilds: {
      immediate: true,
      handler(newVal) {
        if (!newVal) {
          return;
        }
        this.activeGuild = this.activeGuild || newVal[0];
        for (const guild of newVal) {
          if (!guild.icon) {
            continue;
          }
          getColors(guild.icon).then(colors => {
            let first = colors[0];
            let second;
            let distance = 0;

            for (const x of colors) {
              const contrast = chroma.contrast(first, x);
              if (contrast > distance && contrast >= 2) {
                second = x;
                distance = contrast;
              }
            }

            if (!second) {
              first = chroma("black");
              second = chroma("white");
            }

            // this.guildColors.baum = { first, second };
            this.$set(this.guildColors, guild.id, {
              first: first.hex(),
              second: second.hex(),
              darkFirst: first.darken().hex()
            });
            // tmpColors[guild.id] = { first, second };
          });
        }
      }
    }
  },
  methods: {
    ...mapActions(["fetchGuilds"]),
    reload() {
      this.fetchingGuilds = true;
      this.fetchGuilds().finally(() => {
        this.fetchingGuilds = false;
      });
    },
    soundJoinValueChanged(sound, newVal) {
      console.log(`${sound.command} - ${newVal}`);
      if (newVal) {
        this.$set(this.activeGuild, "joinSound", sound.id);
      } else {
        this.$delete(this.activeGuild, "joinSound");
      }
    },
    getPalette(id) {
      return this.guildColors[id] || { first: "#f4f4f4", second: "black" };
    },
    closeUploadSoundDialog() {
      this.addSoundDialog = false;
      this.$refs.addSoundForm.reset();
    },
    submitUploadSoundForm() {
      if (!this.$refs.addSoundForm.validate()) {
        return;
      }
      console.log("file", this.addSoundFormData.file.name);
      let formData = new FormData();
      formData.append("command", this.addSoundFormData.command);
      formData.append("description", this.addSoundFormData.description);
      formData.append("guild", this.activeGuild.id);
      formData.append(
        "file",
        this.addSoundFormData.file,
        this.addSoundFormData.file.name
      );

      //   for (const key of formData.entries()) {
      //     console.log(key[0], key[1]);
      //   }

      axios
        .post("/api/sounds/upload", formData, {
          headers: {
            "Content-Type": "Application/json"
          }
        })
        .then(res => {
          console.log(res);
          this.addSoundDialog = false;
          this.$toast.success(
            `Befehl <b>${this.addSoundFormData.command}</b> erfolgreich gespeichert.`
          );
          this.reload();
        })
        .catch(e => {
          console.log("error");
          this.$toast.error(
            `Befehl konnte nicht gespeichert werden: <b>${e.message}</b>`
          );
        });
      console.log(this.addSoundFormData.file);
    }
  },
  computed: {
    ...mapGetters(["guilds"]),
    activeGuild: {
      get() {
        for (const guild of this.guilds) {
          // if (guild.id === this.activeGuildId) {
          if (guild.id === this.$route.query.guild) {
            return guild;
          }
        }
        return undefined;
      },
      set(value) {
        if (value && value.id) {
          this.$router.push({ query: { guild: value.id } }).catch(() => {});
        }
        // this.activeGuildId = value ? value.id : undefined;
      }
    },
    activeGuildCommands() {
      if (!this.activeGuild) {
        return [];
      }
      return this.activeGuild.sounds.map(sound => sound.command);
    },
    filteredSortedActiveGuildSounds() {
      if (!this.activeGuild) {
        return [];
      }
      let sounds = this.activeGuild.sounds;

      let result = sounds;

      if (this.soundSearchString && this.soundSearchString.length >= 2) {
        const searchString = this.soundSearchString.trim().toLowerCase();
        result = result.filter(sound => {
          return (
            sound.command.toLowerCase().includes(searchString) ||
            sound.description.toLowerCase().includes(searchString)
          );
        });
      }

      let sortMethod = this.sortings[this.sortMethod];

      result = sortMethod.sort(result, this.sortDirection);

      return result;
      // TODO
    },
    getPaginatedSounds() {
      let sounds = this.filteredSortedActiveGuildSounds;
      let paginated = sounds.slice(
        (this.currentSoundPage - 1) * this.soundsPerPage,
        this.currentSoundPage * this.soundsPerPage
      );
      return paginated;
    },
    paginationLength() {
      return Math.ceil(
        this.filteredSortedActiveGuildSounds.length / this.soundsPerPage
      );
    }
  },
  data() {
    return {
      currentSoundPage: 1,
      soundsPerPage: 16,
      soundSearchString: "",

      soundPlaying: false,

      fetchingGuilds: false,
      guildColors: {},
      activeGuildId: undefined,

      addSoundDialog: false,
      addSoundFormData: {
        command: "",
        description: "",
        file: undefined
      },
      sortDirection: 1,
      sortMethod: 0,
      sortings: [
        {
          name: "Alphabetisch",
          sort(list, direction) {
            return list.sort(
              (a, b) => direction * a.command.localeCompare(b.command)
            );
          }
        }
      ],

      validationRules: {
        command: [
          v => !!v || "Befehl wird benötigt",
          v => (v && v.length <= 15) || "Darf nicht länger als 15 Zeichen sein",
          v => (v && v.length >= 3) || "Darf nicht kürzer als 3 Zeichen sein",
          v =>
            !this.activeGuildCommands.includes(v) ||
            "Dieser Befehl existiert bereits",
          v =>
            /^[a-zA-Z0-9äÄöÖüÜß]*$/.test(v) ||
            "Der Befehl enthält ungültige Zeichen"
        ],
        description: [
          v => !!v || "Beschreibung wird benötigt",
          v => (v && v.length <= 60) || "Darf nicht länger als 60 Zeichen sein",
          v => (v && v.length >= 3) || "Darf nicht kürzer als 3 Zeichen sein"
        ]
      }
    };
  }
};
</script>