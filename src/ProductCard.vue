<script>
export default {
    name: 'ProductCard',
    props: {
        products: {
            type: Array
        }
    }
}
</script>
<template>
    <ul class="images-list">
        <li v-for="product in products" :key="product.id" class="images-list-items">
            <a :href="product.thumbnail" target="_blank" id="product-thumbnail">
                <div class="image-wrapper-class">
                    <img class="image-container-1" :src="product.images[0]" :alt="product.title">
                    <img v-if="product.images.length > 1" class="image-container-2" :src="product.images[1]"
                        :alt="product.title">
                    <img v-else class="image-container-2" :src="product.images[0]" :alt="product.title">
                </div>
            </a>
            <p class="product-title">{{ product.title }}</p>
            <p class="product-price">{{ `$${product.price}` }}</p>
        </li>
    </ul>
</template>