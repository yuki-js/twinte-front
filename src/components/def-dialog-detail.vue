<template>
  <!-- 科目詳細画面 -->
  <Dialog :show="show" @close="close()">
    <article v-if="table">
      <!-- 教科名 -->
      <h1>
        <div class="sbj-name">{{ table.lecture_name }}</div>

        <p class="sbj-number">科目番号 {{ table.lecture_code }}</p>
      </h1>

      <!-- 科目詳細 -->
      <h2>
        <i class="material-icons icon">info</i>科目詳細
        <span class="syllabus-btn" @click="syllabus()">
          シラバス
          <i class="material-icons icon">chevron_right</i>
        </span>
      </h2>
      <section class="sbj-detail-wrapper">
        <p class="subject-item">
          担当教員
          <span>{{ table.instructor }}</span>
        </p>
        <p class="subject-item">
          開講時限
          <span>{{ table.module }} {{ table.day }}{{ table.period }}</span>
        </p>
        <p class="subject-item">
          授業教室
          <span v-if="!editableLecture">{{ table.room }}</span>

          <input v-else class="sbj-detail" v-model="editableLecture.room" />
          <!-- → 教室変更 -->
        </p>
      </section>

      <!-- メモ -->
      <h2>
        <i class="material-icons icon">create</i>メモ
        <span class="syllabus-btn" @click="attend()">
          出席
          <i class="material-icons icon">chevron_right</i>
        </span>
      </h2>
      <textarea class="memo" type="text" v-model="localMemo"></textarea>
      <section class="counters-wrapper">
        <div
          v-for="n in 3"
          :key="n"
          :class="{ attend: n === 1, absent: n === 2, late: n === 3 }"
          style="width: 30%"
        >
          <span class="counter-name"
            >{{ atmnb[n - 1] }} {{ atmnbCount[n - 1] }}回</span
          >
          <!-- <+|-> -->
          <div class="counter">
            <span @click="counter(atmnb[n - 1], +1)" class="counter-left"
              >+</span
            >
            <span @click="counter(atmnb[n - 1], -1)" class="counter-right"
              >&#8211;</span
            >
          </div>
        </div>
      </section>
      <div @click="save()" class="save-btn">変更を保存</div>
      <div class="flex">
        <p @click="deleteItem()" class="delete-btn">
          <i class="material-icons icon">delete</i>この科目を削除
        </p>
        <p @click="edit()" class="edit-btn">
          <i class="material-icons icon">edit</i>教室情報を編集
        </p>
      </div>
    </article>
  </Dialog>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator';
import * as Vuex from 'vuex';
import { Period } from '../types';
import { UserLectureEntity } from '../types/server';
import { updateLecture } from '../store/api/timetables';
import cloneDeep from 'lodash/cloneDeep';
import Swal from 'sweetalert2';
import { YEAR } from '../common/config';

@Component({
  components: {
    Dialog: () => import('~/components/global/dialog.vue')
  }
})
export default class Index extends Vue {
  $store!: Vuex.ExStore;

  atmnb = ['出席', '欠席', '遅刻'];
  moduleNum = this.$store.getters['table/moduleNum'];
  localMemo = '';
  localLectureId = '';
  editableLecture: Period | null = null;

  get atmnbCount() {
    return this.userData
      ? [this.userData.attendance, this.userData.absence, this.userData.late]
      : [0, 0, 0];
  }
  get userData() {
    return this.$store.getters['table/userData'];
  }
  get table(): Period | null {
    return this.$store.getters['table/looking'];
  }
  get show(): boolean {
    return this.$store.getters['visible/detail'];
  }

  syllabus() {
    switch (this.table?.lecture_code.substring(0, 2)) {
      case 'GB':
        location.href = `http://www.coins.tsukuba.ac.jp/syllabus/${this.table?.lecture_code}.html`;
      default:
        location.href = `https://kdb.tsukuba.ac.jp/syllabi/${YEAR}/${this.table?.lecture_code}/jpn/#course-title`;
    }
  }
  attend() {
    location.href = 'https://atmnb.tsukuba.ac.jp';
  }
  edit() {
    if (this.editableLecture) {
      this.editableLecture = null;
    } else {
      this.editableLecture = cloneDeep(this.table);
    }
  }
  counter(type: string, num: number) {
    if (!this.userData) {
      return;
    }
    let { attendance, absence, late } = this.userData;
    switch (type) {
      case '出席':
        attendance + num >= 0 ? (attendance += num) : 0;
        break;
      case '欠席':
        absence + num >= 0 ? (absence += num) : 0;
        break;
      case '遅刻':
        late + num >= 0 ? (late += num) : 0;
        break;
    }
    const userData: UserLectureEntity = {
      twinte_lecture_id: this.userData.twinte_lecture_id,
      user_lecture_id: this.userData.user_lecture_id,
      lecture_name: this.userData.lecture_name,
      instructor: this.userData.instructor,
      memo: this.userData.memo,
      attendance,
      absence,
      late
    };
    this.$store.dispatch('table/updatePeriod', { userData });
  }

  deleteItem() {
    Swal.fire({
      title: 'この時間割を削除しますか?',
      showCancelButton: true,
      confirmButtonText: 'はい',
      cancelButtonText: 'いいえ'
    }).then(async result => {
      if (result.value && this.table && this.userData) {
        await this.$store.dispatch('api/deleteTable', {
          table: this.table,
          UserLecture: this.userData
        });

        this.close();
        // → ダイアログを閉じる
      }
    });
  }

  close(): void {
    this.localMemo = '';
    this.editableLecture = null;
    this.$store.commit('visible/chDetail', { display: false });
    this.$store.commit('table/setUserData', { userData: null });
    this.$store.commit('table/setLooking', { period: null });
  }

  async save() {
    if (!this.userData) {
      return;
    }
    const userData: UserLectureEntity = {
      twinte_lecture_id: this.userData.twinte_lecture_id,
      user_lecture_id: this.userData.user_lecture_id,
      lecture_name: this.userData.lecture_name,
      instructor: this.userData.instructor,
      memo: this.localMemo,
      attendance: this.userData.attendance,
      absence: this.userData.absence,
      late: this.userData.late
    };
    await this.$store.dispatch('table/updatePeriod', { userData });
    // → メモの変更

    if (this.editableLecture) {
      await updateLecture(this.editableLecture);
    }
    // → 教室の変更

    this.$store.dispatch('api/login');
    // → 反映

    this.editableLecture = null; // 編集モードをオフに
    this.close(); // 閉じさせる
    Swal.fire('完了', 'メモを保存しました', 'success');
  }

  fetchMemo() {
    setTimeout(() => {
      if (
        this.userData &&
        this.localLectureId !== this.userData.user_lecture_id
      ) {
        this.localMemo = this.userData.memo;
        this.localLectureId = this.userData.user_lecture_id;
      }
      this.fetchMemo();
    }, 1000);
    // リアクティブにできないのは既知のバグ
  }

  mounted() {
    this.$nextTick(() => {
      this.fetchMemo();
    });
  }
}
</script>

<style lang="scss" scoped>
@import '~/assets/css/btn.scss';

//++++++++++++++++++// 以下ダイアログの内容（中身） //+++++++++++++++++//
article {
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  height: 100%;
  width: 100%;
  overflow-y: scroll;
}

/* ボタン・アイコン */
.icon {
  display: inline-flex;
  vertical-align: middle;
  padding-bottom: 0.4vh;
}

/* 科目の情報 */
h1 {
  height: 6rem;
  font-size: 1.7rem;
  color: #555555;
  line-height: 3.4rem;
  font-weight: 500;
  text-overflow: ellipsis;
  border-left: 0.5rem solid #00c0c0;
  // overflow: hidden;
  white-space: nowrap;
  padding-left: 0.8rem;
  margin: 0 0 2vh;
  .sbj-number {
    font-size: 1.2rem;
    color: #c4c4c4;
    font-weight: 400;
    line-height: 2rem;
    margin: 0 0 0.7rem;
  }
}
h2 {
  width: 100%;
  font-size: 1.4rem;
  color: #00c0c0;
  font-weight: 500;
  border-bottom: 0.2vh solid #00c0c0;
  margin: 0;
  i {
    font-size: 2rem;
    margin-right: 1%;
  }
  .syllabus-btn {
    position: absolute;
    right: 0;
    font-size: 1.2rem;
    color: #9a9a9a;
    font-weight: 400;
    line-height: 2.4rem;
    i {
      font-size: 1.6rem;
      margin-right: 0;
    }
  }
}

.sbj-detail-wrapper {
  font-size: 1.4rem;
  padding: 0 2.5vmin;
  margin: 0;
}
.subject-item {
  color: #555555;
  font-weight: 500;

  span {
    padding-left: 5%;
    font-weight: 400;
  }
}

/* メモ */
.memo {
  width: 100%;
  height: 100%;
  min-height: 10vh;
  border: 0.2vh solid #dddddd;
  border-radius: 0.5rem;
  flex-basis: 20vh;
  flex-shrink: 2;
  margin: 1vh 0;
  box-sizing: border-box;
}

/* 出欠 */
.counters-wrapper {
  display: flex;
  justify-content: space-between;
  text-align: center;
  grid-template-columns: repeat(3, 1fr);
  margin-bottom: 2vh;
}
.counter-name {
  line-height: 2.6rem;
  font-size: 1.3rem;
  color: #555555;
}
.counter {
  display: flex;
  justify-content: center;
}
.attend {
  grid-column: 1;
}
.absent {
  grid-column: 2;
}
.late {
  grid-column: 3;
}
.counter-left {
  position: relative;
  height: 3.1rem;
  width: 5rem;
  font-size: 2.2rem;
  color: #00c0c0;
  line-height: 3rem;
  border: 0.2vh solid #00c0c0;
  border-radius: 3.1rem 0 0 3.1rem;
}
.counter-right {
  position: relative;
  left: -0.2vh;
  height: 3.1rem;
  width: 5rem;
  font-size: 2.2rem;
  color: #00c0c0;
  line-height: 2.6rem;
  border: 0.2vh solid #00c0c0;
  border-radius: 0 3.1rem 3.1rem 0;
}
.counter-left:active,
.counter-right:active {
  background-color: #00c0c0;
  color: white;
}

//++++++++++++++++++++++++// 編集・削除ボタン //++++++++++++++++++++++++//
.flex {
  display: flex;
  justify-content: space-between;
}
.delete-btn {
  font-size: 1.3rem;
  color: rgb(255, 98, 98);
  margin: 2vh 0 0;
  i {
    font-size: 2rem;
  }
}
.edit-btn {
  font-size: 1.3rem;
  color: #6678df;
  margin: 2vh 0 0;
  i {
    font-size: 2rem;
  }
}
</style>
