<template>
  <section @click="popUp()">
    <div id="subject" v-if="!table"></div>
    <!-- → 授業が入っていない -->

    <div
      id="subject"
      :style="{
        background: getColor(table.lecture_code)
      }"
      v-else
    >
      <div class="sbj-lectureId">{{ table.lecture_code }}</div>
      <div class="sbj-name">{{ table.lecture_name }}</div>
      <div class="sbj-room">{{ table.room }}</div>
    </div>
    <!-- → 授業が入っている -->
  </section>
</template>

<script lang="ts">
import { Component, Vue, Prop } from "nuxt-property-decorator";
import * as Vuex from "vuex";
import { Period } from "../types/index";

enum Day {
  Sun = "日",
  Mon = "月",
  Tue = "火",
  Wed = "水",
  Thu = "木",
  Fri = "金",
  Sat = "土",
  Unknown = "不明"
}

@Component({})
export default class Index extends Vue {
  $store!: Vuex.ExStore;

  @Prop() day!: number;
  @Prop() period!: number;

  week: string[] = [Day.Mon, Day.Tue, Day.Wed, Day.Thu, Day.Fri];

  get table() {
    const periods = this.$store.getters["api/table"];
    const module = this.module;
    const week = this.week;
    const day = this.day;
    const period = this.period + 1;

    const validPeriod = periods.find(lecture => {
      return (
        lecture.module === module && // module
        week.indexOf(lecture.day) === day && // day
        lecture.period === period // period
      );
    });
    return validPeriod;
  }
  get module() {
    return this.$store.getters["table/module"];
  }
  chDetail(period: Period) {
    this.$store.dispatch("table/setPeriod", { period });
    this.$store.commit("visible/chDetail", { display: true });
  }
  chAdd() {
    this.$store.commit("visible/chAdd", { display: true });
  }
  popUp() {
    if (this.table) {
      this.chDetail(this.table);
    } else {
      this.chAdd();
    }
  }
  /** 授業に応じたテーマ色を返す */
  getColor(number: string): string {
    const char = number.split("")[0];
    switch (char) {
      case "A":
        return "#DEFFF9";
      case "B":
        return "#DEFFF9";
      case "C":
        return "#DEFFF9";
      case "E":
        return "#DEFFF9";
      case "F":
        return "#DEFFF9";
      case "G":
        return "#DEFFF9";
      case "H":
        return "#DEFFF9";
      case "W":
        return "#DEFFF9";
      case "Y":
        return "#DEFFF9";
      case "1":
        return "#FFEEF7";
      case "2":
        return "#F0EBFF";
      case "3":
        return "#FFFCEB";
      default:
        return "";
    }
  }
}
</script>

<style lang="scss" scoped>
$height: calc((100vh - 16.5vh - 6vmin - 12vmin) / 6);
$width: calc(
  (100vw - 8vw /**外枠 */ - 11vw /** 時限 */ - 12vw /** 科目+時限 padding */) /
    5
);
/* 科目 */
#subject {
  color: #555;
  width: $width;
  height: $height;
  padding: 1vmin 1vw;
  word-break: break-all;
  font-style: normal;
  font-weight: 700;
  font-size: 1.3vh;
  line-height: 2vh;
  overflow: hidden;
  &:active {
    transition: all 0.3s;
    filter: brightness(150%);
  }
}
.sbj-lectureId {
  font-weight: 400;
}
.sbj-name {
  font-weight: 700;
}
.sbj-room {
  font-weight: 400;
}
</style>