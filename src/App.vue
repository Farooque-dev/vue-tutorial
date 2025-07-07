<script>
export default {
  data() {
    return {
      products: [],
      originalProducts: [],
      searchQuery: '',
      selectedOptions: 'default',
      totalProducts: 0,
      productsPerPage: 32,
      isVisible: false,
      categories: [],
      showCategories: false,
      showFilterSidebar: false,
      debounceTimer: null,
      selectedCategories: [],
      categoryCounts: {},
      currentPage: 1,
      showMobileSortPopup: false,
      sortOptions: [
        { value: 'default', label: 'Featured' },
        { value: 'price-asc', label: 'Price: Low to High' },
        { value: 'price-desc', label: 'Price: High to Low' },
        { value: 'title-asc', label: 'Alphabetically: A to Z' },
        { value: 'title-desc', label: 'Alphabetically: Z to A' }
      ]
    }
  },
  methods: {
    async fetchData(page) {
      try {
        const response = await fetch(`https://dummyjson.com/products?limit=0`);
        if (!response) {
          throw new Error('No response found');
        }
        const result = await response.json();
        this.products = result.products
        this.totalProducts = result.total;
        console.log(result);
        this.originalProducts = [...this.products];
      } catch (error) {
        console.error(error.message);
      }
    },

    async searchProducts(query) {
      this.selectedOptions = 'default';
      this.selectedCategories = [];
      if (!query) {
        await this.fetchData();
        return;
      }
      try {
        const response = await fetch(`https://dummyjson.com/products/search?q=${query}&limit=0`);
        if (!response) {
          throw new Error('No response found.');
        }
        const result = await response.json();
        this.products = result.products;
        this.currentPage = 1;

        console.log(`query: ${query}`);
        console.log(`result: ${result.products.length}`);
      } catch (error) {
        console.error(error.message);
      }
    },

    sortProducts() {
      try {
        if (this.selectedOptions === 'price-asc') {
          this.products.sort((a, b) => a.price - b.price);
        } else if (this.selectedOptions === 'price-desc') {
          this.products.sort((a, b) => b.price - a.price);
        } else if (this.selectedOptions === 'title-asc') {
          this.products.sort((a, b) => a.title.localeCompare(b.title));
        } else if (this.selectedOptions === 'title-desc') {
          this.products.sort((a, b) => b.title.localeCompare(a.title));
        } else if (this.selectedOptions === 'default' && this.selectedCategories.length === 0) {
          this.products = [...this.originalProducts];
        }
        this.currentPage = 1;
      } catch (error) {
        console.error(error.message);
      }
    },

    scrollToTop() {
      window.scrollTo({
        top: 0,
        behavior: 'smooth'
      });
    },

    returnToTop() {
      this.isVisible = window.scrollY > 200;
    },

    async getCategoriesList() {
      try {
        const response = await fetch('https://dummyjson.com/products/category-list');
        if (!response) {
          throw new Error('No response found.');
        }
        const result = await response.json();
        console.log(result);
        this.categories = result;
        this.showCategories = !this.showCategories;
        const counts = {};
        await Promise.all(result.map(async (cat) => {
          const total = await this.getProductsByCategoryTotal(cat);
          counts[cat] = total || 0;
        }));
        this.categoryCounts = counts;
        console.log(counts);
      } catch (error) {
        console.error(error.message);
      }
    },

    async fetchProductsByCategory(category) {
      try {
        const response = await fetch(`https://dummyjson.com/products/category/${category}`);
        return response;
      } catch (error) {
        return null;
      }
    },

    async getProductsByCategoryTotal(category) {
      try {
        const response = await this.fetchProductsByCategory(category);
        if (!response) {
          throw new Error('No response found.');
        }
        const result = await response.json();
        return result.total;
      } catch (error) {
        console.error(error.message);
        return 0;
      }
    },

    async onCategoryChange() {
      if (this.selectedCategories.length === 0) {
        await this.fetchData();
      } else {
        let allProducts = [];
        let total = 0;
        for (const cat of this.selectedCategories) {
          const response = await this.fetchProductsByCategory(cat);
          if (response.ok) {
            const result = await response.json();
            allProducts = allProducts.concat(result.products);
            total += result.products.length;
          }
        }
        this.products = allProducts;
        this.totalProducts = total;
      }
      this.sortProducts();
    },

    async changePage(page) {
      this.currentPage = page;
      this.scrollToTop();
    },

    debouncedSearch(query) {
      if (this.debounceTimer) {
        clearTimeout(this.debounceTimer);
      }
      this.debounceTimer = setTimeout(() => {
        this.searchProducts(query);
      }, 400);
    }
  },

  mounted() {
    this.fetchData();
    window.addEventListener('scroll', this.returnToTop);
  },

  watch: {
    showFilterSidebar(val) {
      if (val) {
        document.body.style.overflow = 'hidden';
      } else {
        document.body.style.overflow = '';
      }
    },

    showMobileSortPopup(val) {
      if (val) {
        document.body.style.overflow = 'hidden';
      } else {
        document.body.style.overflow = '';
      }
    }
  },

  computed: {
    sortLabel() {
      switch (this.selectedOptions) {
        case 'price-asc':
          return 'Price: Low to High';
        case 'price-desc':
          return 'Price: High to Low';
        case 'title-asc':
          return 'Alphabetically: A to Z';
        case 'title-desc':
          return 'Alphabetically: Z to A';
        default:
          return 'Featured';
      }
    },

    paginatedProducts() {
      const start = (this.currentPage - 1) * this.productsPerPage;
      const end = start + this.productsPerPage;
      return this.products.slice(start, end);
    },

    pageCount() {
      return Math.ceil(this.products.length / this.productsPerPage);
    }
  }
}
</script>

<template>
  <div class="app-container">
    <div class="navbar-container">
      <div class="input-container">
        <input type="text" v-model="searchQuery" @input="debouncedSearch(searchQuery)"
          placeholder="Search for a product...">
      </div>

    </div>
    <div class="filter-sort-container">
      <div class="filter-container">
        <button class="filter-button" @click="showFilterSidebar = true"
          :class="{ 'active-filter': this.selectedCategories.length > 0 }"
          v-if="!showFilterSidebar && !showMobileSortPopup">
          <svg data-v-b856b8c9="" aria-hidden="true" focusable="false" role="presentation" class="filter-icon"
            :class="{ 'active-filter-icon': this.selectedCategories.length > 0 }" viewBox="0 0 64 64">
            <title data-v-b856b8c9="">icon-filter</title>
            <path data-v-b856b8c9=""
              d="M48 42h10m-10 0a5 5 0 1 1-5-5 5 5 0 0 1 5 5ZM7 42h31M16 22H6m10 0a5 5 0 1 1 5 5 5 5 0 0 1-5-5Zm41 0H26">
            </path>
          </svg>
          Filter{{ selectedCategories.length > 0 ? ` (${selectedCategories.length})` : '' }}
        </button>
      </div>
      <div class="sort-container">
        <button class="sort-button" @click="showSortDropdown = !showSortDropdown, showMobileSortPopup = true"
          v-if="!showFilterSidebar && !showMobileSortPopup">
          <span class="sort-label-desktop">{{ sortLabel }}</span>
          <span class="sort-label-mobile">Sort By</span>
          <svg xmlns="http://www.w3.org/2000/svg" width="25px" height="20px" viewBox="0 0 28 24" class="sort-icon">
            <g>
              <path fill="none" d="M0 0h24v24H0z" />
              <path d="M12 15l-4.243-4.243 1.415-1.414L12 12.172l2.828-2.829 1.415 1.414z" fill="#494949" />
            </g>
          </svg>
        </button>
        <div v-if="showMobileSortPopup" class="mobile-sort-popup-overlay" @click="showMobileSortPopup = false">
          <div class="mobile-sort-popup">
            <div class="mobile-sort-title">Sort By</div>
            <ul class="mobile-sort-list">
              <li v-for="option in sortOptions" :key="option.value"
                :class="{ 'selected-sort-option': selectedOptions === option.value }"
                @click="selectedOptions = option.value; sortProducts(); showMobileSortPopup = false"
                class="mobile-sort-item-name">
                {{ option.label }}
              </li>
            </ul>
          </div>
        </div>
      </div>
    </div>
    <div class="total-products">
      <span>{{ this.products.length }} products</span>
    </div>
    <div v-if="showFilterSidebar" class="sidebar-overlay" @click="showFilterSidebar = false">
    </div>
    <div class="filter-sidebar" :class="{ open: showFilterSidebar }">
      <div class="sidebar-header">
        <span>Filter</span>
        <button class="close-button" @click="showFilterSidebar = false">&times;</button>
      </div>
      <div class="sidebar-content">
        <div class="filter-item" style="border-top: none;" @click="getCategoriesList">
          <button id="category">CATEGORY</button>
          <span class="svg-drop-down-icon">
            <svg xmlns="http://www.w3.org/2000/svg" width="12px" height="12px" viewBox="0 0 24 16" fill="none"
              :class="{ 'rotated': showCategories }">
              <path fill-rule="evenodd" clip-rule="evenodd" stroke="#000" stroke-width="2"
                d="M12.7071 14.7071C12.3166 15.0976 11.6834 15.0976 11.2929 14.7071L6.29289 9.70711C5.90237 9.31658 5.90237 8.68342 6.29289 8.29289C6.68342 7.90237 7.31658 7.90237 7.70711 8.29289L12 12.5858L16.2929 8.29289C16.6834 7.90237 17.3166 7.90237 17.7071 8.29289C18.0976 8.68342 18.0976 9.31658 17.7071 9.70711L12.7071 14.7071Z"
                fill="none" />
            </svg>
          </span>
        </div>
        <ul v-if="showCategories" class="category-list">
          <li v-for="cat in categories" :key="cat" class="filter-list-items">
            <label class="filter-label">
              <input type="checkbox" class="input-checkbox" v-model="selectedCategories" @change="onCategoryChange"
                :value="cat" />
              <span class="checkmark">
                <span class="checkmark-inner"></span>
              </span>
              <span class="filter-name" :class="{ 'filter-name-selected': selectedCategories.includes(cat) }">{{ cat }}
                ({{ categoryCounts[cat] }})</span>
            </label>
          </li>
        </ul>
      </div>
    </div>

    <div v-if="products.length > 0" class="main-content">
      <div class="left-content">
        <ul class="filter-list">
          <div class="filter-item" style="border-top: none;" @click="getCategoriesList">
            <button id="category">CATEGORY</button>
            <span class="svg-drop-down-icon">
              <svg xmlns="http://www.w3.org/2000/svg" width="12px" height="12px" viewBox="0 0 24 16" fill="none"
                :class="{ 'rotated': showCategories }">
                <path fill-rule="evenodd" clip-rule="evenodd" stroke="#000" stroke-width="2"
                  d="M12.7071 14.7071C12.3166 15.0976 11.6834 15.0976 11.2929 14.7071L6.29289 9.70711C5.90237 9.31658 5.90237 8.68342 6.29289 8.29289C6.68342 7.90237 7.31658 7.90237 7.70711 8.29289L12 12.5858L16.2929 8.29289C16.6834 7.90237 17.3166 7.90237 17.7071 8.29289C18.0976 8.68342 18.0976 9.31658 17.7071 9.70711L12.7071 14.7071Z"
                  fill="none" />
              </svg>
            </span>
          </div>
          <ul v-if="showCategories" class="category-list">
            <li v-for="cat in categories" :key="cat" class="filter-list-items">
              <label class="filter-label">
                <input type="checkbox" class="input-checkbox" v-model="selectedCategories" @change="onCategoryChange"
                  :value="cat" />
                <span class="checkmark">
                  <span class="checkmark-inner"></span>
                </span>
                <span class="filter-name" :class="{ 'filter-name-selected': selectedCategories.includes(cat) }">{{ cat
                  }} ({{ categoryCounts[cat] }})</span>
              </label>
            </li>
          </ul>
        </ul>
      </div>
      <div class="right-content">

        <div class="top-right-container">
          <span v-if="searchQuery" class="products-count">
            Showing {{ products.length }} {{ products.length === 1 ? 'result' : 'results' }} for "{{ searchQuery }}"
          </span>
          <span v-else class="products-count">{{ this.totalProducts }} products</span>
          <div class="sort-by-container">
            <select id="sort-by" @change="sortProducts()" v-model="selectedOptions">
              <option value="default">Featured</option>
              <option value="price-asc">Price: Low to High</option>
              <option value="price-desc">Price: High to Low</option>
              <option value="title-asc">Alphabetically: A to Z</option>
              <option value="title-desc">Alphabetically: Z to A</option>
            </select>
          </div>
        </div>
        <ul class="images-list">
          <li v-for="product in paginatedProducts" :key="product.id" class="images-list-items">
            <a :href="product.thumbnail" target="_blank" id="product-thumbnail">
              <div class="image-wrapper-class">
                <img class="image-container-1" :src="product.images[0]" alt="">
                <img v-if="product.images.length > 1" class="image-container-2" :src="product.images[1]" alt="">
                <img v-else class="image-container-2" :src="product.images[0]" alt="">
              </div>
              <p class="product-title">{{ product.title }}</p>
              <p class="product-price">{{ `$${product.price}` }}</p>
            </a>
          </li>
        </ul>
      </div>
    </div>
    <div v-else class="loading-container">
      <p>No products found.</p>
    </div>
    <div class="pagination" v-if="this.products.length > this.productsPerPage">
      <div v-show="this.products.length > this.productsPerPage" v-for="page in pageCount" :key="page" class="page-item"
        @click="changePage(page)" :class="['page-item', { 'active-page': currentPage === page }]">
        {{ page }}
      </div>
    </div>
    <div class="scroll-to-top" v-show="isVisible">
      <button class="scroll-to-top" @click="scrollToTop()">Scroll to Top</button>
    </div>
  </div>
</template>

<style>
@import "./style.css";
</style>
