<template>
  <v-container class="statistics" fluid>
    <div v-if="playerstats.length > 0">
      <v-container
        v-for="(playerMapStats, index) in playerstats"
        :key="playerMapStats[0].id"
      >
        <v-container class="mapinfo" fluid>
          <div
            class="text-subtitle-2 mapInfo"
            v-if="arrMapString[index] != null"
            align="center"
          >
            {{ arrMapString[index].score }} - {{ arrMapString[index].map }}
          </div>
          <div
            class="text-subtitle-2 mapInfo"
            v-if="
              arrMapString[index] != null && arrMapString[index].start != null
            "
            align="center"
          >
            {{ arrMapString[index].start }}
          </div>
          <div
            class="text-subtitle-2 mapInfo"
            v-if="
              arrMapString[index] != null && arrMapString[index].end != null
            "
            align="center"
          >
            {{ arrMapString[index].end }}
          </div>
          <div
            class="text-subtitle-2 mapInfo"
            v-if="
              arrMapString[index] != null && arrMapString[index].demo != null
            "
            align="center"
          >
            <v-btn
              small
              color="secondary"
              :href="apiUrl + '/demo/' + arrMapString[index].demo"
            >
              {{ $t("PlayerStats.Download") }}
            </v-btn>
          </div>
          <div
            class="text-subtitle-2 mapInfo"
            v-if="
              arrMapString[index] != null && arrMapString[index].end == null
            "
            align="left"
          >
            <div class="text-caption" v-if="!isFinished">
              {{ $t("PlayerStats.RefreshData", { sec: countDownTimer }) }}
              <v-btn
                x-small
                color="secondary"
                @click="refreshStats"
                :disabled="countDownTimer >= 55"
              >
                {{ $t("PlayerStats.RefreshForce") }}
              </v-btn>
            </div>
            <div v-else />
          </div>
        </v-container>
        <v-data-table
          item-key="id"
          class="elevation-1"
          :items-per-page="12"
          :loading="isLoading"
          :loading-text="$t('misc.LoadText')"
          :headers="headers"
          :items="playerMapStats"
          sort-by="kills"
          sort-desc
          ref="PlayerStats"
          group-by="Team"
          show-group-by
          hide-default-footer
          show-expand
        >
          <template v-slot:item.name="{ item }">
            <router-link :to="{ path: '/user/' + item.steam_id }">
              {{ item.name }}
            </router-link>
          </template>
          <template v-slot:item.Team="{ item }">
            {{ item.Team.slice(2) }}
          </template>
          <template v-slot:expanded-item="{ item }" class="text-center">
            <td :colspan="headers.length">
              <v-data-table
                item-key="id"
                class="elevation-1"
                :headers="additionalHeaders"
                hide-default-footer
                dense
                :key="item.id"
                :items="[item]"
                disable-sort
                :colspan="headers.length"
              />
            </td>
          </template>
        </v-data-table>
      </v-container>
    </div>
    <div v-else align="center">
      <strong>
        {{ $t("PlayerStats.NoStatFound") }}
      </strong>
    </div>
  </v-container>
</template>

<script>
export default {
  props: {
    match_id: Number
  },
  data() {
    return {
      playerstats: [],
      isLoading: true,
      arrMapString: [{}],
      playerInterval: -1,
      countDownTimer: 60,
      allowRefresh: false,
      timeoutId: -1,
      isFinished: false,
      apiUrl: process.env?.VUE_APP_G5V_API_URL || "/api"
    };
  },
  created() {
    // Template will contain v-rows/etc like on main Team page.
    this.GetMapPlayerStats();
    this.getMapString();
  },
  beforeDestroy() {
    if (!this.isFinished) {
      if (this.timeoutId != -1) clearInterval(this.timeoutId);
      if (this.playerInterval != -1) clearInterval(this.playerInterval);
    }
  },
  computed: {
    headers() {
      return [
        {
          text: this.$t("PlayerStats.User"),
          align: "start",
          sortable: true,
          value: "name",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.Kills"),
          value: "kills",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.Deaths"),
          value: "deaths",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.Assists"),
          value: "assists",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.FlashbangAssists"),
          value: "flashbang_assists",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.EnemyFlash"),
          value: "enemies_flashed",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.FriendFlash"),
          value: "friendlies_flashed",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.Rating"),
          value: "rating",
          groupable: false
        },
        {
          text: this.$t("PlayerStats.TeamName"),
          value: "Team",
          align: "right"
        },
        {
          text: "",
          value: "data-table-expand",
          groupable: false,
          align: "end"
        }
      ];
    },
    additionalHeaders() {
      return [
        {
          text: this.$t("PlayerStats.Suicides"),
          value: "suicides"
        },
        {
          text: this.$t("PlayerStats.ADR"),
          value: "adr"
        },
        {
          text: this.$t("PlayerStats.UtilDamage"),
          value: "util_damage"
        },
        {
          text: this.$t("PlayerStats.KnifeKills"),
          value: "knife_kills"
        },
        {
          text: this.$t("PlayerStats.BombPlants"),
          value: "bomb_plants"
        },
        {
          text: this.$t("PlayerStats.BombDefuses"),
          value: "bomb_defuses"
        },
        {
          text: this.$t("PlayerStats.Headshot") + "%",
          value: "hsp"
        },
        {
          text: this.$t("PlayerStats.KDR"),
          value: "kdr"
        },
        {
          text: this.$t("PlayerStats.FPR"),
          value: "fpr"
        },
        {
          text: this.$t("PlayerStats.KAST"),
          value: "kast"
        },
        {
          text: this.$t("PlayerStats.ContribScore"),
          value: "contribution_score"
        },
        {
          text: this.$t("PlayerStats.MVP"),
          value: "mvp"
        }
      ];
    }
  },
  methods: {
    async GetMapPlayerStats() {
      try {
        let res = await this.GetPlayerStats(this.match_id);
        let getMatchTeamIds = await this.GetMatchData(this.match_id);
        if (typeof res == "string") return;
        let allMapIds = [];
        let totalMatchTeam = [];
        let allTeamIds = [];
        res.filter(item => {
          let i = allMapIds.findIndex(x => x == item.map_id);
          let j = allTeamIds.findIndex(x => x == item.team_id);
          if (i <= -1) allMapIds.push(item.map_id);
          if (j <= -1) allTeamIds.push(item.team_id);
          return null;
        });
        allMapIds.forEach(map_id => {
          totalMatchTeam.push(
            res.filter(stats => {
              return stats.map_id == map_id;
            })
          );
        });
        this.playerstats = totalMatchTeam;
        await this.playerstats.forEach((matchStats, idx) => {
          matchStats.forEach(async (player, pIdx) => {
            if (player.roundsplayed > 0) {
              let getRating = this.GetRating(
                player.kills,
                player.roundsplayed,
                player.deaths,
                player.k1,
                player.k2,
                player.k3,
                player.k4,
                player.k5
              );
              let adr = this.GetADR(player);
              let hsp = this.GetHSP(player);
              let kdr = this.GetKDR(player);
              let fpr = this.GetFPR(player);
              let teamNum = player.team_id == getMatchTeamIds.team1_id ? 1 : 2;
              let newName =
                player.team_id == getMatchTeamIds.team1_id
                  ? getMatchTeamIds.team1_string
                  : getMatchTeamIds.team2_string;
              this.$set(
                this.playerstats[idx][pIdx],
                "Team",
                teamNum + " " + newName
              );
              this.$set(this.playerstats[idx][pIdx], "rating", getRating);
              this.$set(this.playerstats[idx][pIdx], "adr", adr);
              this.$set(this.playerstats[idx][pIdx], "hsp", hsp);
              this.$set(this.playerstats[idx][pIdx], "kdr", kdr);
              this.$set(this.playerstats[idx][pIdx], "fpr", fpr);
            }
          });
        });
        if (getMatchTeamIds.end_time != null) this.isFinished = true;
        if (!this.isFinished && this.playerInterval == -1) {
          this.playerInterval = setInterval(async () => {
            this.isLoading = true;
            this.GetMapPlayerStats();
            this.getMapString();
            this.countDownTimer = 60;
          }, 60000);
          this.timeoutId = setInterval(() => {
            this.countDownTimer--;
          }, 1000);
        }
      } catch (error) {
        console.log("Our error: " + error);
      } finally {
        this.isLoading = false;
      }
      return;
    },
    async getMapString() {
      try {
        let mapStats = await this.GetMapStats(this.match_id);
        if (typeof mapStats == "string") return;
        mapStats.forEach((singleMapStat, index) => {
          this.arrMapString[index] = {};
          this.arrMapString[index].score =
            "Score: " +
            singleMapStat.team1_score +
            " " +
            this.GetScoreSymbol(
              singleMapStat.team1_score,
              singleMapStat.team2_score
            ) +
            " " +
            singleMapStat.team2_score;
          this.arrMapString[index].start =
            "Map Start: " + new Date(singleMapStat.start_time).toLocaleString();
          this.arrMapString[index].end =
            singleMapStat.end_time == null
              ? null
              : "Map End: " + new Date(singleMapStat.end_time).toLocaleString();
          this.arrMapString[index].map = "Map: " + singleMapStat.map_name;
          this.arrMapString[index].demo = singleMapStat.demoFile;
        });
      } catch (error) {
        console.log("String error " + error);
      }
    },
    async refreshStats() {
      clearInterval(this.timeoutId);
      clearInterval(this.playerInterval);
      this.countDownTimer = 60;
      this.playerInterval = setInterval(async () => {
        this.isLoading = true;
        this.GetMapPlayerStats();
        this.getMapString();
        this.countDownTimer = 60;
      }, 60000);
      this.timeoutId = setInterval(() => {
        this.countDownTimer--;
      }, 1000);
      this.GetMapPlayerStats();
      this.getMapString();
      return;
    }
  }
};
</script>
