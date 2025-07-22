<script>
import ProductCard from './ProductCard.vue';
import debounce from 'debounce';
export default {
  data() {
    return {
      products: [],
      originalProducts: [],
      searchQuery: '',
      debouncedSearchFn: null,
      selectedOptions: 'default',
      totalProducts: 0,
      productsPerPage: 16,
      isVisible: false,
      categories: [],
      showCategories: false,
      showFilterSidebar: false,
      debounceTimer: null,
      selectedCategory: '',
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
  components: {
    ProductCard
  },
  methods: {
    async fetchData(page) {
      try {
        this.searchQuery = '';
        const skip = this.skipProducts(page || 1);

        if (!this.selectedCategory && this.selectedOptions === 'default') {
          const response = await fetch(`https://dummyjson.com/products?limit=${this.productsPerPage}&skip=${skip}`);

          if (!response.ok) {
            throw new Error('No response found');
          }

          const result = await response.json();
          this.products = result.products
          this.totalProducts = result.total;
          this.originalProducts = [...this.products];
          return this.products;
        } else if (this.selectedCategory) {
          await this.sortProducts(this.selectedCategory, skip);
          this.originalProducts = [...this.products];
          return this.products;
        } else {
          const response = await fetch(`https://dummyjson.com/products?limit=${this.productsPerPage}&skip=${skip}`);
          if (!response.ok) throw new Error('No response found');
          const result = await response.json();
          this.products = result.products;
          this.totalProducts = result.total;
          this.originalProducts = [...this.products];
          return this.products;
        }
      } catch (error) {
        console.error(error.message);
      }
    },

    async searchProducts(query, page = 1) {

      if (!query) {
        this.currentPage = 1;
        await this.fetchData();
        return;
      } else {
        try {
          this.selectedOptions = 'default';
          this.selectedCategory = '';
          const skip = this.skipProducts(page);
          const response = await fetch(`https://dummyjson.com/products/search?q=${query}&limit=${this.productsPerPage}&skip=${skip}`);
          if (!response.ok) {
            throw new Error('No response found.');
          }
          const result = await response.json();
          this.products = result.products;
          this.totalProducts = result.total;
        } catch (error) {
          console.error(error.message);
        }
      }
    },

    async onSortChange() {
      this.currentPage = 1;
      const skip = this.skipProducts(this.currentPage);
      await this.sortProducts(this.selectedCategory, skip, this.searchQuery);
    },

    async sortProducts(category, skip, search) {
      this.currentPage = this.currentPage || 1;
      skip = skip !== undefined ? skip : this.skipProducts(this.currentPage);

      const baseURL = 'https://dummyjson.com/products';
      const params = new URLSearchParams({
        limit: this.productsPerPage,
        skip
      });

      let url = baseURL;

      if (category) {
        url += `/category/${category}`;
      } else if (search) {
        url += `/search`;
        params.set('q', search);
      }

      const [sortField, sortOrder] = this.selectedOptions.split('-');
      if (sortField && sortField !== 'default') {
        params.set('sortBy', sortField);
        if (sortOrder) {
          params.set('order', sortOrder);
        }
      }

      const fullUrl = `${url}?${params.toString()}`;

      try {
        const response = await fetch(fullUrl);
        if (!response.ok) throw new Error('No response found.');
        const result = await response.json();
        this.products = result.products;
        this.totalProducts = result.total;
      } catch (error) {
        console.error(error.message);
      }
    }
    ,

    scrollToTop() {
      window.scrollTo({
        top: 0,
        behavior: 'smooth'
      });
    },

    returnToTop() {
      this.isVisible = window.scrollY > 100;
    },

    async getCategoriesList() {
      try {
        const response = await fetch('https://dummyjson.com/products/category-list');
        if (!response.ok) {
          throw new Error('No response found.');
        }
        const result = await response.json();
        this.categories = result;
        this.showCategories = !this.showCategories;
        const counts = {};
        await Promise.all(result.map(async (cat) => {
          const total = await this.getProductsByCategoryTotal(cat);
          counts[cat] = total || 0;
        }));
        this.categoryCounts = counts;
      } catch (error) {
        console.error(error.message);
      }
    },

    async fetchProductsByCategory(category, skip) {
      try {
        const response = await fetch(`https://dummyjson.com/products/category/${category}?limit=${this.productsPerPage}&skip=${skip}`);
        if (!response.ok) {
          throw new Error('No response found.');
        }
        return response;
      } catch (error) {
        return null;
      }
    },

    async getProductsByCategoryTotal(category) {
      try {
        const response = await this.fetchProductsByCategory(category, 0);
        if (!response.ok) {
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
      this.currentPage = 1;
      await this.fetchData();
    },

    async changePage(page) {
      this.currentPage = page;
      this.scrollToTop();
      const skip = this.skipProducts(this.currentPage);
      if (this.searchQuery || this.selectedOptions !== 'default') {
        await this.sortProducts(this.selectedCategory, skip, this.searchQuery);
      }
      else if (this.selectedCategory) {
        await this.sortProducts(this.selectedCategory, skip)
      }
      else {
        await this.fetchData(page);
      }
    },

    async resetFilters() {
      this.selectedCategory = '';
      this.currentPage = 1;
      await this.fetchData();
      if (this.selectedOptions === 'default') {
        this.products = [...this.originalProducts];
      } else {
        this.sortProducts(this.selectedCategory, this.skipProducts(this.currentPage));
      }
    },
    skipProducts(page) {
      return (page - 1) * this.productsPerPage;
    }
  },

  mounted() {
    this.fetchData();
    window.addEventListener('scroll', this.returnToTop);
    this.debouncedSearchFn = debounce(async (query) => {
      await this.searchProducts(query);
    }, 400);
  },

  watch: {
    searchQuery(newQuery) {
      this.debouncedSearchFn(newQuery);
    },

    showFilterSidebar(val) {
      if (val) {
        document.body.classList.add('no-background-scroll');
      } else {
        document.body.classList.remove('no-background-scroll');
      }
    },

    showMobileSortPopup(val) {
      if (val) {
        document.body.classList.add('no-background-scroll');
      } else {
        document.body.classList.remove('no-background-scroll');
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

    pageCount() {
      return Math.ceil(this.totalProducts / this.productsPerPage);
    }
  }
}
</script>

<template>
  <div class="app-container">
    <div class="navbar-container">
      <div class="input-container">
        <input type="text" v-model="searchQuery" placeholder="Search for a product...">
      </div>

    </div>
    <div class="filter-sort-container">
      <div class="filter-container">
        <span class="filter-button" @click="showFilterSidebar = true"
          :class="{ 'active-filter': this.selectedCategory }" v-if="!showFilterSidebar && !showMobileSortPopup">
          <svg aria-hidden="true" focusable="false" role="presentation" class="filter-icon"
            :class="{ 'active-filter-icon': !this.selectedCategory }" viewBox="0 0 64 64">
            <title>icon-filter</title>
            <path
              d="M48 42h10m-10 0a5 5 0 1 1-5-5 5 5 0 0 1 5 5ZM7 42h31M16 22H6m10 0a5 5 0 1 1 5 5 5 5 0 0 1-5-5Zm41 0H26">
            </path>
          </svg>
          Filter
        </span>
      </div>
      <div class="sort-container">
        <div class="sort-button" @click="showMobileSortPopup = true" v-if="!showFilterSidebar && !showMobileSortPopup">
          <span class="sort-label-desktop">{{ sortLabel }}</span>
          <span class="sort-label-mobile">Sort By</span>
          <svg xmlns="http://www.w3.org/2000/svg" width="25px" height="20px" viewBox="0 0 28 24" class="sort-icon">
            <g>
              <path fill="none" d="M0 0h24v24H0z" />
              <path d="M12 15l-4.243-4.243 1.415-1.414L12 12.172l2.828-2.829 1.415 1.414z" fill="#494949" />
            </g>
          </svg>
        </div>
        <div v-if="showMobileSortPopup" class="mobile-sort-popup-overlay" @click="showMobileSortPopup = false">
          <div class="mobile-sort-popup">
            <div class="mobile-sort-title">Sort By</div>
            <ul class="mobile-sort-list">
              <li v-for="option in sortOptions" :key="option.value"
                :class="{ 'selected-sort-option': selectedOptions === option.value }"
                @click="selectedOptions = option.value; showMobileSortPopup = false; onSortChange()"
                class="mobile-sort-item-name">
                {{ option.label }}
              </li>
            </ul>
          </div>
        </div>
      </div>
    </div>
    <div v-if="this.totalProducts > 0" class="total-products">
      <span>{{ this.totalProducts > 1 ? this.totalProducts + ' products' : this.totalProducts + ' product'
        }}</span>
    </div>
    <div v-if="showFilterSidebar" class="sidebar-overlay" @click="showFilterSidebar = false">
    </div>
    <div class="filter-sidebar" :class="{ open: showFilterSidebar }">
      <div class="sidebar-header">
        <span>Filter</span>
        <button v-if="selectedCategory" @click="resetFilters" :class="{ 'isReset': selectedCategory }"
          style="margin: 10px 0 10px 0;">
          Reset
        </button>
        <button class="close-button" @click="showFilterSidebar = false">&times;</button>
      </div>
      <div class="sidebar-content">
        <div class="filter-item" @click="getCategoriesList">
          <span id="category">CATEGORY</span>
          <span class="svg-drop-down-icon">
            <svg xmlns="http://www.w3.org/2000/svg" width="14px" height="20px" viewBox="0 0 28 16" fill="none"
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
              <input type="radio" class="input-checkbox" v-model="selectedCategory" @change="onCategoryChange"
                :value="cat" />
              <span class="checkmark">
                <span class="checkmark-inner"></span>
              </span>
              <span class="filter-name" :class="{ 'filter-name-selected': selectedCategory === cat }">{{ cat }}
                ({{ categoryCounts[cat] }})</span>
            </label>
          </li>
        </ul>
      </div>
    </div>

    <div v-if="products && products.length > 0" class="main-content">
      <div class="left-content">
        <ul class="filter-list">
          <div class="filter-item" style="border-top: none;" @click="getCategoriesList">
            <span id="category">CATEGORY</span>
            <button v-show="this.selectedCategory" @click.stop="resetFilters"
              :class="{ 'isReset': this.selectedCategory }">Reset</button>
            <span class="svg-drop-down-icon">
              <svg xmlns="http://www.w3.org/2000/svg" width="14px" height="20px" viewBox="0 0 28 16" fill="none"
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
                <input type="radio" class="input-checkbox" v-model="selectedCategory" @change="onCategoryChange"
                  :value="cat" />
                <span class="checkmark">
                  <span class="checkmark-inner"></span>
                </span>
                <span class="filter-name" :class="{ 'filter-name-selected': selectedCategory === cat }">{{ cat
                }} ({{ categoryCounts[cat] }})</span>
              </label>
            </li>
          </ul>
        </ul>
      </div>
      <div class="right-content">

        <div class="top-right-container">
          <span v-if="searchQuery" class="products-count">
            Showing {{ totalProducts }} {{ totalProducts === 1 ? 'result' : 'results' }} for "{{ searchQuery }}"
          </span>
          <span v-else class="products-count">{{ this.totalProducts }} products</span>
          <div class="sort-by-container">
            <select id="sort-by" @change="onSortChange" v-model="selectedOptions">
              <option value="default">Featured</option>
              <option value="price-asc">Price: Low to High</option>
              <option value="price-desc">Price: High to Low</option>
              <option value="title-asc">Alphabetically: A to Z</option>
              <option value="title-desc">Alphabetically: Z to A</option>
            </select>
          </div>
        </div>
        <ProductCard :products="products" />
      </div>
    </div>
    <div v-else class="loading-container">
      <img src="../download_1.png" alt="No results found image" class="loading-image">
      <div class="loading-message">{{ `No products found for "${searchQuery}"` }}</div>
    </div>
    <div class="pagination" v-if="this.totalProducts > this.productsPerPage">
      <div v-show="this.totalProducts > this.productsPerPage" v-for="page in pageCount" :key="page" class="page-item"
        @click="changePage(page)" :class="['page-item', { 'active-page': currentPage === page }]">
        {{ page }}
      </div>
    </div>
    <div class="scroll-to-top" v-show="isVisible">
      <button class="scroll-to-top-btn" @click="scrollToTop()"><svg xmlns="http://www.w3.org/2000/svg"
          xmlns:xlink="http://www.w3.org/1999/xlink" fill="#000000" height="20px" width="20px" version="1.1"
          id="Layer_1" viewBox="0 0 511.735 511.735" xml:space="preserve">
          <g>
            <g>
              <path
                d="M508.788,371.087L263.455,125.753c-4.16-4.16-10.88-4.16-15.04,0L2.975,371.087c-4.053,4.267-3.947,10.987,0.213,15.04    c4.16,3.947,10.667,3.947,14.827,0l237.867-237.76l237.76,237.76c4.267,4.053,10.987,3.947,15.04-0.213    C512.734,381.753,512.734,375.247,508.788,371.087z" />
            </g>
          </g>
        </svg></button>
    </div>
  </div>
</template>

<style>
@import "./style.css";
</style>
