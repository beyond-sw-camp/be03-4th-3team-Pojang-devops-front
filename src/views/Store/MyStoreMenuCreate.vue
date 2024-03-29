<template>
  <div>
    <button @click="openModal" class="bg-blue-500 text-white py-2 px-4 rounded">메뉴 등록</button>
    <!-- 메뉴 리스트 -->
    <div class="menu-list">
      <div v-for="menu in menus" :key="menu.menuId" class="card ">
        <div @click="toggleOptions(menu.menuId)" style="cursor: pointer" class="card-body flex flex-col mt-2 mb-2 border-b-4 relative">
          <div class="flex mb-3">
            <img :src="menu.imageUrl" class="card-img-top w-[100px] h-[100px] mr-4 mb-2" />
            <div>
              <p class="text-lg font-bold text-gray-800 mb-2">메뉴 이름 : {{ menu.menuName }}</p>
              <p class="text-md text-gray-800 mb-1">메뉴 설명 : {{ menu.info }}</p>
              <p class="text-md text-gray-800 mb-1">금액 : {{ menu.price }}원</p>
            </div>
          </div>
          <div v-if="visibleOptions.has(menu.menuId)" class="flex flex-col">
            <div v-for="optionGroup in menu.optionGroups" :key="optionGroup.optionGroupId" class="mt-1 p-3 border-2 rounded-md block text-md font-bold text-gray-800">
              {{ optionGroup.optionGroupName }}
              <div class="mt-2">
                <p v-for="option in optionGroup.options" :key="option.optionId" class="mb-1 text-sm text-left text-gray-600 rounded-md">
                  {{ option.optionName }} : {{ option.price }} 원
                </p>
              </div>
            </div>
          </div>

          <!-- 옵션 추가 버튼 -->
          <button @click="openOptionModal(menu)" class="bg-green-500 text-white py-2 px-4 rounded absolute top-0 right-0 mt-2">옵션 추가</button>
        </div>
      </div>
    </div>
    <!-- 메뉴 등록 모달 창 -->
    <div v-if="isModalOpen"
      class="fixed top-0 left-0 w-full h-full flex justify-center items-center bg-black bg-opacity-50">
      <div class="bg-white p-8 rounded-lg w-96">
        <div class="flex justify-end">
          <button @click="closeModal"
            class="hover:bg-gray-500 hover:text-white text-gray-500 py-1 px-2 rounded ml-2">X</button>
        </div>
        <label for="name" class="block mb-2">메뉴명:</label>
        <input type="text" maxlength="10" id="name" v-model="name" class="w-full mb-4 p-2 border border-gray-300 rounded-md" />
        <label for="menuInfo" class="block mb-2">메뉴 소개:</label>
        <input type="text" maxlength="30" id="menuInfo" v-model="menuInfo" class="w-full mb-4 p-2 border border-gray-300 rounded-md">
        <label for="price" class="block mb-2">가격:</label>
        <input type="text" id="price" v-model="price" class="w-full mb-4 p-2 border border-gray-300 rounded-md" />
        <label for="image" class="block mb-2">메뉴 이미지</label>
        <input type="file" id="image" @change="handleFileUpload" accept="image/*"
          class="w-full mb-4 p-2 border border-gray-300 rounded-md" />
        <div class="flex justify-end"> <!-- Move buttons to the right -->
          <button @click="registerMenu" class="bg-green-500 he text-white py-2 px-4 rounded">메뉴 등록</button>
        </div>
      </div>
    </div>
    <!-- 옵션 추가 모달 창 -->
    <div v-if="isOptionModalOpen"
      class="fixed top-0 left-0 w-full h-full flex justify-center items-center bg-black bg-opacity-50 overflow-y-auto">
      <div class="bg-white p-8 rounded-lg w-3/5 h-80 overflow-y-auto">
        <div class="flex justify-end">
          <button @click="closeOptionModal"
            class="hover:bg-gray-500 hover:text-white text-gray-500 py-1 px-2 rounded ml-2">X</button>
        </div>

        <!-- 옵션 카테고리 입력 -->
        <label class="block mr-2 text-xl whitespace-no-wrap w-56">{{ selectedMenu.menuName }}</label>
        <div class="mb-4 flex items-center border-b border-gray-500 p-4 w-full">
          <label for="optionGroup" class="block mr-2 text-xl whitespace-no-wrap w-56">옵션 카테고리:</label>
          <input type="text" id="optionGroup" v-model="optionGroup"
            class="w-full p-2 border border-gray-300 rounded-md" />
        </div>

        <!-- 옵션 목록 -->
        <div v-for="(option, optionIndex) in menuOptions" :key="optionIndex" class="mb-4 border p-4 rounded">
          <div class="flex items-center mb-2 w-full">
            <label for="option" class="block mr-2 w-32">옵션명:</label>
            <input type="text" v-model="option.name" class="w-full p-2 border border-gray-300 rounded-md" />
          </div>

          <div class="flex items-center mb-2 w-full">
            <label for="optionPrice" class="block mr-2 w-32">옵션가격:</label>
            <input type="text" v-model="option.price" class="w-full p-2 border border-gray-300 rounded-md" />
          </div>
        </div>

        <!-- 옵션 추가 및 완료 버튼 -->
        <div class="flex justify-end">
          <button @click="addOption" class="bg-green-500 text-white py-2 px-4 rounded mr-2">옵션 추가</button>
          <button @click="completeOption" class="bg-blue-500 text-white py-2 px-4 rounded">완료</button>
        </div>
      </div>
    </div>
  </div>
</template>


<script>
import axios from 'axios';

export default {
  props: ['storeId'],
  data() {
    return {
      isModalOpen: false,
      name: "",
      menuInfo: "",
      price: "",
      image: null,
      option: "",
      optionPrice: "",
      isOptionModalOpen: false,
      optionGroup: "",
      selectedMenu: null, // 선택된 메뉴
      menuOptions: [], // 선택된 메뉴의 옵션 목록
      menus: [],
      visibleOptions: new Set(),
    };
  },
  async created() {
    try {
      const token = localStorage.getItem('token');
      const headers = { Authorization: `Bearer ${token}` };
      const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/api/stores/${this.storeId}/menus`, { headers });
      console.log(response.data.result);
      this.menus = response.data.result;
    } catch (error) {
      console.error(error);
    }
  },
  methods: {
    toggleOptions(menuId) {
      if (this.visibleOptions.has(menuId)) {
        this.visibleOptions.delete(menuId); // 펼쳐진 토글을 닫음
      } else {
        this.visibleOptions.add(menuId); // 숨겨진 토글을 펼침
      }
    },
    openModal() {
      this.name = ""
      this.menuInfo = ""
      this.price = ""
      this.isModalOpen = true;
    },
    closeModal() {
      this.isModalOpen = false;
    },
    handleFileUpload(event) {
      this.image = event.target.files[0];
      if (this.image) {
        this.menuImage = URL.createObjectURL(this.image);
      }
    },
    async registerMenu() {
      // 메뉴 등록 로직을 여기에 구현하세요
      if (confirm("메뉴를 등록하시겠습니까?")) {
        const registerData = new FormData();
        registerData.append("name", this.name);
        registerData.append("menuInfo", this.menuInfo);
        registerData.append("price", this.price);
        if (this.image) {
          registerData.append("image", this.image);
        }
        let headers = {
          'Content-Type': 'multipart/form-data'
        };
        try {
          const token = localStorage.getItem('token');
          if (token == null) {
            alert("로그인이 필요합니다.");
            this.$router.push({ name: "Login" });
            return;
          }
          headers.Authorization = `Bearer ${token}`;
          const response = await axios.post(`${process.env.VUE_APP_API_BASE_URL}/api/stores/${this.storeId}/menus`, registerData, { headers });
          console.log(response.data.result);
          alert("메뉴를 등록하였습니다.");
          this.closeModal();
          this.menus.push(response.data.result);
        } catch (error) {
          console.error(error);
          alert(error.response.data.message);
        }
      }
    },
    openOptionModal(menu) {
      this.selectedMenu = menu;
      this.menuOptions = [];
      this.option = ""
      this.optionPrice = ""
      this.optionGroup = ""
      this.isOptionModalOpen = true;
    },
    closeOptionModal() {
      this.isOptionModalOpen = false;
      this.selectedMenu = null;
      this.menuOptions = [];
    },
    addOption() {
      this.menuOptions.push({ name: "", price: "" });
    },
    async completeOption() {
      if (confirm("메뉴 옵션을 등록하시겠습니까?")) {
        try {
          const token = localStorage.getItem('token');
          const headers = { Authorization: `Bearer ${token}` };
          if (token == null) {
            alert("로그인이 필요합니다.");
            this.$router.push({ name: "Login" });
            return;
          }
          const registerOptionInfo = {
            name: this.optionGroup,
            options: this.menuOptions,
          };
          const response = await axios.post(`${process.env.VUE_APP_API_BASE_URL}/api/stores/${this.storeId}/menus/${this.selectedMenu.menuId}/options`, registerOptionInfo, { headers });
          console.log(response.data.result);
          alert("메뉴 옵션을 등록했습니다.");
          this.closeOptionModal();
        } catch (error) {
          console.error(error);
          alert(error.response.data.message);
        }
      }
    },
  },
};
</script>

<style>
.options-container {
  transition: all 0.3s ease;
  padding: 10px;
  background-color: #f9f9f9;
  border-radius: 8px;
  margin-top: 10px;
}
.option-group {
  margin-bottom: 10px;
}
.option-group-title {
  font-weight: bold;
  margin-bottom: 5px;
}
.option {
  padding: 5px 0;
}
</style>
