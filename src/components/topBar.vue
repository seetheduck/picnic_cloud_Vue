<template>
  <header>
    <div id="upper-nav">
      <div id="logo">
        <router-link to="/">
          <img src="../image/logo.svg" alt="picniCloud" />
        </router-link>
      </div>
      <nav>
        <ul>
          <li v-if="!isLoggedIn" >
            <router-link to="/login">로그인</router-link>
          </li>
          <li v-if="isLoggedIn" class="nav-item">
            <router-link to="/myPage">마이페이지</router-link>
            <div class="logout-and-notification">
            <a @click="logout" href="#">로그아웃</a>
            <!-- 알림 아이콘 클릭 시 모달 창 열기 -->
            <span @click="toggleChatModal" class="notification-icon">🔔</span>
            </div>
            <!-- 모달 창 (알림 아이콘 아래) -->
            <div v-if="isChatModalVisible" class="chat-modal">
              <h3>채팅 목록</h3>
              <div
                v-for="chatRoom in chatRooms"
                :key="chatRoom.no"
                class="chat-room-item"
                @click="enterChatRoom(chatRoom.chatRoomNo)"
              >
                <!-- 보낸 사람을 별도의 태그로 분리 -->
                <span class="chat-room-otherId">{{ chatRoom.otherId }}</span>
                <!-- 메시지 내용을 별도의 태그로 분리 -->
                <span class="chat-room-message">{{ chatRoom.lastMessage }}</span>
                <span class="chat-room-time">{{
                  formatLastMessageTime(chatRoom.lastMessageTime)
                }}</span>
                <!-- 나가기 버튼 -->
                <button
                  class="close-button"
                  @click.stop="leaveChatRoom(chatRoom.chatRoomNo)"
                >
                  &times;
                </button>
              </div>
              <div v-if="chatRooms.length === 0">채팅방이 없습니다</div>
              <button @click="toggleChatModal" class="closeModal">닫기</button>
            </div>
          </li>
        </ul>
      </nav>
    </div>
    <nav class="bottom-nav">
      <ul>
        <li>
          <router-link to="/place">
            <img src="../image/navIcon/iconPlace.png" alt="테마별 장소" /><br />
            <span class="nav-text">테마별 장소</span>
          </router-link>
        </li>
        <li>
          <router-link to="/fleaMarketMain">
            <img src="../image/navIcon/iconMarket.png" alt="피클마켓" /><br />
            <span class="nav-text">피클마켓</span>
          </router-link>
        </li>
        <li>
          <router-link to="/map">
            <img src="../image/navIcon/iconMap.png" alt="키즈존" /><br />
            <span class="nav-text">키즈존</span>
          </router-link>
        </li>
        <li>
          <router-link to="/bookMain">
            <img src="../image/navIcon/iconBook.png" alt="어린이도서" /><br />
            <span class="nav-text">어린이영어도서</span>
          </router-link>
        </li>
      </ul>
    </nav>
  </header>
</template>

<script>
import { ref, computed, onMounted } from "vue";
import { useStore } from "vuex";
import { useRouter } from "vue-router";
import apiClient from "@/api/api.js";
import jwt_decode from "jwt-decode";
import { toast } from 'vue3-toastify'; // toast 함수 임포트
import 'vue3-toastify/dist/index.css'; // 토스트 스타일 임포트
import "./topBar.css";

export default {
  setup() {
    const store = useStore();
    const router = useRouter();
    const chatRooms = ref([]);
    const isChatModalVisible = ref(false);

    const isLoggedIn = computed(() => store.getters.isLoggedIn);

    const toggleChatModal = () => {
      if (!isLoggedIn.value) {
        toast.error("로그인이 필요합니다.");
        router.push("/login"); // 로그인 페이지로 이동
      } else {
        isChatModalVisible.value = !isChatModalVisible.value;
        if (isChatModalVisible.value) {
          getChatRoomList(); // 모달이 열릴 때 채팅 목록을 가져옴
        }
      }
    };

    const getChatRoomList = async () => {
  const token = localStorage.getItem("token");
  if (!token) {
    console.error("토큰이 없습니다.");
    return;
  }

  const decodedToken = jwt_decode(token);
  const currentTime = Math.floor(Date.now() / 1000);

  if (decodedToken.exp < currentTime) {
    console.error("토큰이 만료되었습니다.");
    localStorage.removeItem("token");
    router.push("/login");
    return;
  }

  try {
    const response = await apiClient.get(`/api/chatList`, {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    });
    
    // 채팅 목록을 마지막 메시지 시간 기준으로 정렬 (최근 메시지가 위로 오게)
    chatRooms.value = response.data.sort((a, b) => {
      const timeA = new Date(a.lastMessageTime);
      const timeB = new Date(b.lastMessageTime);
      return timeB - timeA; // 시간 비교 (내림차순)
    });

    console.log("채팅 목록 조회 성공:", response.data);
  } catch (error) {
    console.error("채팅 목록 조회 실패", error);
  }
};

    const enterChatRoom = (chatRoomNo) => {
      router.push(`/listToChatRoom/${chatRoomNo}`);
      toggleChatModal();
    };

    const formatLastMessageTime = (lastMessageTime) => {
      const now = new Date();
      const messageTime = new Date(lastMessageTime);
      const diffInSeconds = Math.floor((now - messageTime) / 1000);
      const diffInMinutes = Math.floor(diffInSeconds / 60);
      const diffInHours = Math.floor(diffInMinutes / 60);

      if (diffInSeconds < 60) {
        return `${diffInSeconds}초 전`;
      } else if (diffInMinutes < 60) {
        return `${diffInMinutes}분 전`;
      } else if (diffInHours < 24) {
        return `${diffInHours}시간 전`;
      } else {
        const options = {
          year: "numeric",
          month: "2-digit",
          day: "2-digit",
          hour: "2-digit",
          minute: "2-digit",
        };
        return messageTime.toLocaleDateString("ko-KR", options);
      }
    };

    const logout = () => {
      store.dispatch("logout");
      toast.success("로그아웃 되었습니다.");
      router.push("/");
    };

    const leaveChatRoom = async (chatRoomNo) => {
      const confirmLeave = confirm("정말로 채팅방을 나가시겠습니까?");
      if (confirmLeave) {
        try {
          const token = localStorage.getItem("token");
          await apiClient.delete(`/api/leave/${chatRoomNo}`, {
            headers: {
              Authorization: `Bearer ${token}`,
            },
          });
          // 나간 후 목록에서 해당 채팅방을 제거
          chatRooms.value = chatRooms.value.filter(
            (room) => room.chatRoomNo !== chatRoomNo
          );
          console.log(`채팅방 ${chatRoomNo} 나가기 성공`);
        } catch (error) {
          console.error("채팅방 나가기에 실패했습니다.", error);
        }
      }
    };

    onMounted(() => {
      const token = localStorage.getItem("token");
      if (!token) {
        store.dispatch("logout");
      }
    });

    return {
      isLoggedIn,
      toggleChatModal,
      logout,
      chatRooms,
      enterChatRoom,
      formatLastMessageTime,
      isChatModalVisible,
      leaveChatRoom,
    };
  },
};
</script>

