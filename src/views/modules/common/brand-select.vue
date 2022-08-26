<template>
  <div>
    <el-select placeholder="请选择" v-model="brandId" filterable clearable>
      <el-option
        v-for="item in brands"
        :key="item.brandId"
        :label="item.brandName"
        :value="item.brandId"
      ></el-option>
    </el-select>
  </div>
</template>

<script>

export default {
  components: {},
  props: {},
  data() {
    return {
      catId: 0,
      brands: [
        {
          label: "a",
          value: 1
        }
      ],
      brandId: "",
      subscribe: null
    };
  },
  watch: {
    brandId(val) {
      this.PubSub.publish("brandId", val);
    }
  },
  methods: {
    getCatBrands() {
      this.$http({
        url: this.$http.adornUrl("/product/categorybrandrelation/brands/list"),
        method: "get",
        params: this.$http.adornParams({
          catId: this.catId
        })
      }).then(({ data }) => {
        this.brands = data.data;
      });
    }
  },
  mounted() {
    //监听三级分类消息的变化
    this.subscribe = this.PubSub.subscribe("catPath", (msg, val) => {
      this.catId = val[val.length - 1];
      this.getCatBrands();
    });
  },
  beforeDestroy() {
    this.PubSub.unsubscribe(this.subscribe); //销毁订阅
  }
};
</script>
<style scoped>
</style>
