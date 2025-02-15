<template>
  <div id="placeInfo" class="container">
    <!-- 장소 기본설명 -->
    <div v-if="place">
      <h1>{{ place.name }}</h1>
      <div class="content">
        <div class="place-image">
          <img
            v-if="place.image"
            :src="getImageUrl(place.image)"
            alt="placeImage"
          />
          <div v-else>
            <img
              :src="getDefaultImageUrl(place.placeType)"
              alt="defaultImage"
              v-if="getDefaultImageUrl(place.placeType)"
            />
            <!-- 기본 이미지도 없을 경우 텍스트 표시 -->
            <p v-else>해당 장소 유형에 대한 이미지가 없습니다</p>
          </div>
        </div>
        <!-- 장소 세부 정보 -->
        <div class="place-details">
          <p>
            {{
              place.address ? `주소: ${place.address}` : "주소 정보가 없습니다"
            }}
          </p>
          <p>
            📞
            {{
              place.tel ? `전화번호: ${place.tel}` : "전화번호 정보가 없습니다"
            }}
          </p>
          <p>⭐{{ place.point.toFixed(1) }}</p>
          <!-- 좋아요 버튼 -->
          <p>
            <button @click="toggleLike" class="like-button">
              {{ userHasLiked ? "❤️ 좋아요 취소" : "🩶좋아요" }}
            </button>
            ({{ placelikeCount }})
          </p>
          <!-- 목록으로 돌아가기 버튼 -->
          <p>
            <button @click="goBackToList" class="back-button">
              목록으로 돌아가기
            </button>
          </p>
        </div>
      </div>

      <!-- 탭 버튼 -->
      <ul class="tabs">
        <!-- 첫 번째 탭: 상세 정보 -->
        <li
          class="tab-button"
          :class="{ active: activeTab === 'tab1' }"
          @click="activeTab = 'tab1'"
        >
          상세정보
        </li>
        <!-- 두 번째 탭: 리뷰 -->
        <li
          class="tab-button"
          :class="{ active: activeTab === 'tab2' }"
          @click="activeTab = 'tab2'"
        >
          리뷰( {{ place.reviewCount }} )
        </li>
      </ul>

      <!-- 탭 내용 -->
      <div class="tab-content">
        <!-- 첫 번째 탭 내용: 장소 상세 정보 -->
        <div
          id="tab1"
          class="tab-pane"
          :class="{ active: activeTab === 'tab1' }"
        >
          <p>
            <span class="bold-text">장소 설명</span><br />
            {{ place.description ? place.description : "추가 정보가 없습니다" }}
          </p>
          <br />
          <p>
            <span class="bold-text">입장료</span><br />
            {{
              place.entranceFee ? place.entranceFee : "입장료 정보가 없습니다"
            }}
          </p>
          <br />
          <p>
            <span class="bold-text">운영시간</span><br />
            {{
              place.operationTime
                ? place.operationTime
                : "운영시간 정보가 없습니다"
            }}
          </p>
        </div>

        <!-- 두 번째 탭 내용: 리뷰 -->
        <div
          id="tab2"
          class="tab-pane"
          :class="{ active: activeTab === 'tab2' }"
        >
          <div>
            <!-- 리뷰 작성 -->
            <h2 v-if="isLoggedIn">리뷰 작성</h2>
            <form v-if="isLoggedIn" @submit.prevent="submitReview">
              <!-- 별점 선택 -->
              <div class="rating">
                <span
                  v-for="n in 5"
                  :key="n"
                  @click="setRating(n)"
                  @mouseover="onStarHover(n)"
                  @mouseleave="resetRating"
                  :class="{
                    star: true,
                    active: n <= currentRating || n <= hoverRating,
                  }"
                >
                  ★
                </span>
              </div>
              <p>{{ currentRating }} 점</p>

              <!-- 리뷰 작성 -->
              <div class="reviewWrite">
                <label for="reviewContents">리뷰 내용 (최대 300자):</label>
                <textarea
                  id="reviewContents"
                  v-model="newReview.contents"
                  maxlength="300"
                  required
                ></textarea>
                <p>{{ newReview.contents.length }} / 300</p>
              </div>

              <button type="submit">리뷰 제출</button>
            </form>
            <p v-else>
              리뷰를 작성하려면
              <router-link to="/login">로그인</router-link>하세요.
            </p>

            <!-- 리뷰 목록 -->
            <ul class="reviewList">
              <li v-for="review in reviews" :key="review.no">
                <div class="review-header">
                  <!-- 왼쪽 정렬: 작성자, 작성일, 신고, 수정, 삭제 -->
                  <div class="review-left">
                    <h4>
                      <span v-if="review.id">{{ review.id }}</span>
                    </h4>
                    <span v-if="review.createDate"
                      >| 작성일: {{ formatDate(review.createDate) }}</span
                    >
                    |
                    <a
                      v-if="isLoggedIn"
                      href="javascript:void(0)"
                      @click="goToReportPage(review.no, placeNo)"
                      class="action-link"
                    >
                      신고
                    </a>
                    <span
                      v-if="isLoggedIn && review.id === loggedInUserId"
                      class="edit-links"
                    >
                      <a
                        href="javascript:void(0)"
                        @click="editReview(review)"
                        class="action-link"
                        >수정</a
                      >
                      <a
                        href="javascript:void(0)"
                        @click="deleteReview(review.no)"
                        class="action-link"
                        >삭제</a
                      >
                    </span>
                  </div>
                  <!-- 오른쪽 정렬: 평점 -->
                  <div class="review-right">
                    <span v-if="review.point !== null"
                      >평점: {{ review.point }}점</span
                    >
                  </div>
                </div>
                <div class="reviewText">
                  <p v-if="review.contents">{{ review.contents }}</p>
                  <p v-if="review.blocked">이 리뷰는 차단되었습니다.</p>

                  <!-- 좋아요 버튼 및 좋아요 수 -->
                  <div class="like-button">
                    <button @click="toggleReviewLike(review)">
                      {{ review.userHasLiked ? "❤️좋아요" : "🩶좋아요" }}
                    </button>
                    ({{ review.likeCnt }})
                  </div>
                </div>
              </li>
            </ul>
            <!-- ConfirmToast 컴포넌트 -->
    <ConfirmToast
      v-if="showToast"
      message="정말로 이 리뷰를 삭제하시겠습니까?"
      :onConfirm="deleteReview"
      :onCancel="cancelDelete"
    />

            <!-- 페이지네이션 -->
            <div v-if="totalPages > 1" class="pagination">
              <a
                href="javascript:void(0)"
                @click="previousPage"
                :class="{ disabled: currentPage === 0 }"
                >&lt;</a
              >
              <span>페이지 {{ currentPage + 1 }} / {{ totalPages }}</span>
              <a
                href="javascript:void(0)"
                @click="nextPage"
                :class="{ disabled: currentPage === totalPages - 1 }"
                >&gt;</a
              >
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 장소 정보가 없는 경우 메시지 표시 -->
    <div v-else>
      <p>해당 번호의 장소는 존재하지 않습니다</p>
    </div>
  </div>
</template>

<script>
import apiClient from "@/api/api.js";
import { getUserIdFromToken } from "@/utils/auth";
import { toast } from "vue3-toastify"; // toast 함수 임포트
import "../css/placeInfo.css";

export default {
  
  name: "PlaceInfo",
  
  data() {
    return {
      userId: "",
      place: null,
      activeTab: "tab1",
      placelikeCount: 0,
      userHasLiked: false, // 사용자 좋아요 상태를 관리
      reviewText: "", // 리뷰 텍스트를 저장할 데이터 속성
      token: localStorage.getItem("token"), // 사용자 로그인 여부 확인
      reviews: [],
      currentPage: 0,
      totalPages: 0,
      placeNo: null, // 쿼리스트링에서 가져오기 위해 기본값 설정
      newReview: {
        // 새로운 리뷰 데이터를 담을 객체
        contents: "",
        point: null, // 평점 필드를 활성화
        id: null, // 로그인한 사용자의 ID
      },
      editMode: false, // 리뷰 수정 모드를 위한 플래그
      editReviewNo: null, // 수정할 리뷰의 no 저장
      isLoggedIn: false, // 로그인 상태 체크
      currentRating: 0, // 현재 선택된 별점
      hoverRating: 0, // 마우스 오버 시 강조된 별점
    };
  },
  created() {
    this.placeNo = this.$route.query.placeNo; // 쿼리스트링에서 placeNo 값 가져오기
    this.checkLoginStatus(); // 로그인 상태 확인
  },

  methods: {
    initializeUser() {
      this.userId = getUserIdFromToken(this.token);
    },
    goBackToList() {
      this.$router.back();
    },

    // 장소 정보 및 좋아요 상태 초기화
    async showInfo() {
      const no = this.$route.query.placeNo;
      if (no) {
        let query = `/places/${no}`;
        if (this.userId) {
          query += `?userId=${this.userId}`;
        }
        await apiClient
          .get(query)
          .then((res) => {
            this.place = res.data;
            // 좋아요 수를 가져오는 메서드 호출
            this.fetchPlaceLikeCount(no);
            // 사용자가 좋아요를 눌렀는지 확인하는 메서드 호출
            this.checkLikeStatus(no);
          })
          .catch((err) => {
            console.log("장소 정보: ", err);
          });
      } else {
        console.log("No place number provided in the URL.");
      }
    },
    // 장소 좋아요 수 가져오기
    async fetchPlaceLikeCount(placeNo) {
      await apiClient
        .get(`/places/${placeNo}/likes-count`)
        .then((res) => {
          this.placelikeCount = res.data;
        })
        .catch((err) => {
          console.log("좋아요 수 가져오기 오류: ", err);
        });
    },
    // 장소 좋아요 상태 확인 및 토글 처리
    async checkLikeStatus() {
      if (!this.place || !this.place.no) return;
      await apiClient
        .get(`/places/${this.place.no}/likes-status`, {
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
          params: {
            userId: this.userId,
          },
        })
        .then((res) => {
          this.userHasLiked = res.data.liked; // 서버가 반환하는 값에 맞춰 처리
          this.placelikeCount = res.data.likeCount; // 서버에서 업데이트된 좋아요 수를 받음
        })
        .catch((err) => {
          console.log("좋아요 상태 확인 오류: ", err);
        });
    },

    // 장소 좋아요 버튼 클릭
    async toggleLike() {
      if (!this.token) {
        toast.error("로그인이 필요합니다.");
        this.$router.push({
          path: "/login",
          query: { redirect: this.$route.fullPath },
        });
        return;
      }
      if (!this.place || !this.place.no) return;

      try {
        const res = await apiClient.post(
          `/places/${this.place.no}/likes-toggle`,
          null,
          {
            headers: {
              Authorization: `Bearer ${this.token}`,
            },
            params: {
              userId: this.userId,
            },
          }
        );

        this.userHasLiked = res.data.liked; // 좋아요 상태 토글
        this.placelikeCount = res.data.likeCount; // 좋아요 수 업데이트

        // 좋아요 상태 변경에 대한 단일 토스트 알림
        toast.success("좋아요 상태가 변경되었습니다.");
      } catch (err) {
        console.log("좋아요 토글 오류: ", err);
      }
    },

    // 장소 이미지관리
    getImageUrl(imagePath) {
      return imagePath ? `/${imagePath}` : "";
    },
    getDefaultImageUrl(placeType) {
      const defaultImageUrls = {
        "공공형 키즈카페": require("@/image/placePic/공공형 키즈카페_1.jpg"),
        아동서점: require("@/image/placePic/아동서점_1.jpg"),
        자연휴양림: require("@/image/placePic/자연휴양림_1.jpg"),
        캠핑: require("@/image/placePic/캠핑_1.jpg"),
      };
      return defaultImageUrls[placeType] || "";
    },
    // 로그인 상태 확인 메서드
    checkLoginStatus() {
      const token = localStorage.getItem("token");
      if (token) {
        this.isLoggedIn = true;
        this.loggedInUserId = getUserIdFromToken(); // 토큰에서 사용자 ID 추출
      } else {
        this.isLoggedIn = false;
        console.log("로그인 상태 아님");
      }
    },
    // 리뷰 목록 가져오기
    fetchReviews(page = 0) {
      if (!this.placeNo) {
        console.error("placeNo 값이 없습니다.");
        return;
      }

      apiClient
        .get(`/reviews/${this.placeNo}`, {
          params: {
            page: page,
            size: 5,
          },
        })
        .then((response) => {
          // 리뷰 객체에 초기 속성 추가
          this.reviews = response.data.content.map((review) => ({
            ...review,
            userHasLiked: false,
            likeCnt: 0,
          }));

          this.currentPage = response.data.page.number;
          this.totalPages = response.data.page.totalPages;

          // 각 리뷰에 대해 좋아요 상태 및 좋아요 수 확인
          this.reviews.forEach((review) => {
            this.checkReviewLikeStatus(review);
          });

          // 응답 데이터 확인 (디버깅 용도)
          console.log(this.reviews);
        })
        .catch((error) => {
          console.error(
            "리뷰를 가져오는 중 오류 발생:",
            error.response?.data || error.message
          );
        });
    },
    //리뷰 좋아요 상태 확인 및 좋아요 수 가져오기
    async checkReviewLikeStatus(review) {
      await apiClient
        .get(`/reviews/${review.no}/likes-status`, {
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
          params: {
            userId: this.userId,
          },
        })
        .then((res) => {
          review.userHasLiked = res.data.liked; // 서버가 반환하는 좋아요 상태
          review.likeCnt = res.data.likeCount; // 서버에서 반환하는 좋아요 수
        })
        .catch((err) => {
          console.log("리뷰 좋아요 상태 확인 오류: ", err);
        });
    },
    //리뷰 좋아요 버튼 클릭
    async toggleReviewLike(review) {
      if (!this.token) {
        toast.error("로그인이 필요합니다.");
        this.$router.push({
          path: "/login",
          query: { redirect: this.$route.fullPath },
        });
        return;
      }

      try {
        const res = await apiClient.post(
          `/reviews/${review.no}/likes-toggle`,
          null,
          {
            headers: {
              Authorization: `Bearer ${this.token}`,
            },
            params: {
              userId: this.userId,
            },
          }
        );

        review.userHasLiked = res.data.liked; // 서버가 반환한 토글된 좋아요 상태
        review.likeCnt = res.data.likeCount; // 서버에서 업데이트된 좋아요 수 받음

        // 좋아요 상태 변경에 대한 단일 토스트 알림
        toast.success("좋아요 상태가 변경되었습니다.");
      } catch (err) {
        console.log("리뷰 좋아요 토글 오류: ", err);
      }
    },

    // 리뷰 좋아요 수 가져오기
    async fetchReviewLikeCount(review) {
      await apiClient
        .get(`/reviews/${review.no}/likes-count`)
        .then((res) => {
          review.likeCnt = res.data; // 서버에서 반환하는 좋아요 수
        })
        .catch((err) => {
          console.log("리뷰 좋아요 수 가져오기 오류: ", err);
        });
    },

    formatDate(date) {
      return new Date(date).toLocaleString(); // 날짜와 시간을 로컬 형식으로 출력
    },
    //페이징
    previousPage() {
      if (this.currentPage > 0) {
        this.fetchReviews(this.currentPage - 1);
      }
    },
    nextPage() {
      if (this.currentPage < this.totalPages - 1) {
        this.fetchReviews(this.currentPage + 1);
      }
    },
    // 별점 설정 메서드
    setRating(rating) {
      this.currentRating = rating;
      this.newReview.point = rating; // 리뷰의 별점에 설정된 값을 저장
    },
    // 마우스 오버 시 별 강조
    onStarHover(rating) {
      this.hoverRating = rating;
    },
    // 마우스가 별에서 나갈 때
    resetRating() {
      this.hoverRating = 0;
    },
    // 리뷰 제출 메서드
    submitReview() {
      if (!this.newReview.contents || !this.newReview.point) {
        toast.error("리뷰 내용과 별점을 모두 입력해주세요.");
        return;
      }

      // placeNo 추가 확인
      this.newReview.placeNo = this.placeNo; // 프론트엔드에서 장소 번호 설정
      // 로그인한 사용자 ID를 newReview 객체에 추가
      this.newReview.id = this.loggedInUserId;

      // 수정 모드일 경우
      if (this.editMode) {
        this.updateReview();
      } else {
        // 신규 리뷰 작성
        apiClient
          .post(`/reviews/${this.placeNo}`, this.newReview)
          .then(() => {
            toast.success("리뷰가 성공적으로 등록되었습니다.");
            this.fetchReviews(); // 리뷰 목록을 다시 불러옵니다.
            this.showInfo();
            this.newReview.contents = ""; // 리뷰 작성 후 입력 폼을 초기화합니다.
            this.newReview.point = null;
            this.currentRating = 0; // 별점도 초기화
          })
          .catch((error) => {
            console.error("리뷰 등록 중 오류 발생:", error);
          });
      }
    },
    // 리뷰 수정 모드로 전환
    editReview(review) {
      this.editMode = true;
      this.editReviewNo = review.no;
      this.newReview.contents = review.contents;
      this.currentRating = review.point; // 현재 별점도 수정에 맞게 설정
      this.newReview.placeNo = review.placeNo; // 리뷰와 연결된 placeNo를 추가
      this.newReview.id = this.loggedInUserId; // 수정 모드에서도 id 추가
    },
    // 리뷰 수정 요청
    updateReview() {
      console.log(this.newReview); // newReview의 값 확인

      apiClient
        .put(`/reviews/${this.editReviewNo}`, this.newReview, {
          headers: {
            Authorization: `Bearer ${localStorage.getItem("token")}`, // 토큰을 헤더로 전달
          },
        })
        .then(() => {
          toast.success("리뷰가 성공적으로 수정되었습니다.");
          this.fetchReviews();
          this.cancelEditMode(); // 수정 후 초기화
        })
        .catch((error) => {
          console.error("리뷰 수정 중 오류 발생:", error);
        });
    },
    // 수정 모드 취소
    cancelEditMode() {
      this.editMode = false;
      this.editReviewId = null;
      this.newReview.contents = "";
      this.newReview.point = null;
      this.currentRating = 0;
    },

    // 리뷰 삭제 요청
    deleteReview(reviewNo) {
      //const userId = this.loggedInUserId; // 로그인된 사용자 ID

      if (confirm("정말로 이 리뷰를 삭제하시겠습니까?")) {
        apiClient
          .delete(`/reviews/${reviewNo}`, {
            headers: {
              Authorization: `Bearer ${localStorage.getItem("token")}`,
            },
          })
          .then(() => {
            toast.success("리뷰가 성공적으로 삭제되었습니다.");
            this.fetchReviews(); // 삭제 후 리뷰 목록 갱신
            this.showInfo();
          })
          .catch((error) => {
            console.error("리뷰 삭제 중 오류 발생:", error); // 오류 로그 출력
            // 오류 응답 내용을 출력해서 더 많은 정보 얻기
            console.error(error.response?.data || error.message);
            console.log(localStorage.getItem("token"));
          });
      }
    },

    goToReportPage(reviewNo, placeNo) {
      this.$router.push({
        name: "reviewReport",
        params: {
          reviewNo: reviewNo,
          placeNo: placeNo,
        },
      });
    },
  },
  mounted() {
    this.initializeUser();
    this.showInfo();
    this.fetchReviews(); // 리뷰 목록 가져오기
  },
};
</script>
