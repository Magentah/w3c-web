<template>
  <v-data-table hide-default-footer :headers="headers" :items="gameModeStatsCombined" mobile-breakpoint="400" :items-per-page="-1">
    <template v-for="h in headers" v-slot:[`header.${h.text}`]="{ header }">
      <v-tooltip top v-bind:key="h.text">
        <template v-slot:activator="{ on }">
          <span v-on="on">{{ header.text }}</span>
        </template>
        <span style="white-space: pre-line">{{ header.tooltip }}</span>
      </v-tooltip>
    </template>
    <template v-slot:body="{ items }">
      <tbody>
        <tr v-for="item in items" :key="item.id">
          <td class="cell d-flex justify-center align-center">
            <span>{{ $t("gameModes." + EGameMode[item.gameMode]) }}</span>
            <race-icon style="display: inline; padding-left: 10px" :race="item.race" />
          </td>
          <td class="number-text text-start cell">
            <div class="text-center">
              <span class="won">{{ item.wins }}</span>
              -
              <span class="lost">{{ item.losses }}</span>
            </div>
            <div class="sub-value">{{ (item.winrate * 100).toFixed(1) }}%</div>
          </td>
          <td class="number-text text-end cell">
            <div class="text-center">
              {{ item.rank !== 0 ? item.mmr : "-" }}
            </div>
            <div v-if="item.rank !== 0" class="sub-value">
              {{ getTopPercent(item) }}
            </div>
          </td>
          <td class="number-text text-center cell" style="min-width: 100px">
            <level-progress v-if="item.rank !== 0" :rp="item.rankingPoints"></level-progress>
            <div v-else>-</div>
          </td>
        </tr>
      </tbody>
    </template>
  </v-data-table>
</template>

<script lang="ts">
import { computed, defineComponent } from "vue";
import { useI18n } from "vue-i18n-bridge";
import { TranslateResult } from "vue-i18n";
import { AT_modes } from "@/mixins/GameModesMixin";
import { EGameMode } from "@/store/types";
import { ModeStat } from "@/store/player/types";
import RaceIcon from "@/components/player/RaceIcon.vue";
import LevelProgress from "@/components/ladder/LevelProgress.vue";
import isEmpty from "lodash/isEmpty";

interface ModeStatsGridHeader {
  text: TranslateResult;
  align: string;
  sortable: boolean;
  tooltip: TranslateResult;
}

export default defineComponent({
  name: "ModeStatsGrid",
  components: {
    RaceIcon,
    LevelProgress,
  },
  props: {
    stats: {
      type: Array<ModeStat>,
      required: true,
    },
  },
  setup(props) {
    const { t } = useI18n();

    const isAtMode = (mode: EGameMode): boolean => AT_modes().includes(mode);

    const gameModeStatsCombined = computed<ModeStat[]>(() => {
      const AT_stats = props.stats.filter((g) => isAtMode(g.gameMode));

      if (AT_stats.length === 0) return sortByName(props.stats);

      // Arrange AT stats in separate arrays for each AT mode.
      const AT_arranged = AT_modes().map((AT_mode) => AT_stats.filter((modeStat) => AT_mode == modeStat.gameMode));

      // Filter out AT modes that haven't been played, sort by ranking points and get the stat with the highest rank.
      const topAtStats = AT_arranged.filter((modeStats) => !isEmpty(modeStats))
        .map((modeStats) => modeStats.sort((modeStat) => modeStat.rankingPoints))
        .map((modeStats) => modeStats.filter((modeStat, index) => index === 0))
        .flat();

      // Filter out the old AT stats.
      const modifiedStats = props.stats.filter((g) => !isAtMode(g.gameMode));

      // Add the new AT stats.
      topAtStats.forEach((modeStat) => {
        modifiedStats.push(modeStat);
      });

      return sortByName(modifiedStats);
    });

    function sortByName(mode: ModeStat[]): ModeStat[] {
      return mode.sort((a, b) => (EGameMode[a.gameMode] < EGameMode[b.gameMode] ? -1 : 1));
    }

    function getTopPercent(modeStat: ModeStat): string {
      if (modeStat.rank <= 0) {
        return "";
      }
      const quantilePerc = modeStat.quantile * 100;
      const topPerc = 100 - quantilePerc;

      if (topPerc > 90) {
        return "";
      }

      return `${t("common.top")} ${Math.max(topPerc, 0.1).toFixed(1)}%`;
    }

    const headers: ModeStatsGridHeader[] = [
      {
        text: t("components_player_modestatsgrid.mode"),
        align: "center",
        sortable: false,
        tooltip: t("components_player_modestatsgrid.mode"),
      },
      {
        text: t("components_player_modestatsgrid.winloss"),
        align: "center",
        sortable: false,
        tooltip: t("components_player_modestatsgrid.winloss"),
      },
      {
        text: t("components_player_modestatsgrid.mmr"),
        align: "center",
        sortable: false,
        tooltip: t("components_player_modestatsgrid.mmr"),
      },
      {
        text: t("components_player_modestatsgrid.level"),
        align: "center",
        sortable: false,
        tooltip: t("components_player_modestatsgrid.leveldesc"),
      },
    ];

    return {
      headers,
      gameModeStatsCombined,
      EGameMode,
      getTopPercent,
    };
  },
});
</script>

<style lang="scss" scoped>
.sub-value {
  font-size: 11px;
  border-top: 2px solid #122a42;
  text-align: center;
}

.theme--light {
  .sub-value {
    border-top: 2px solid rgb(205, 205, 205);
  }
}

.tooltip-inner {
  white-space: pre-line;
}

.cell {
  white-space: nowrap;
}
</style>
