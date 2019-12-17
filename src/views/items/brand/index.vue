<template>
  <div class="goods_brand">
    <div class="brand-info">
      <div class="name">
        <img class="img"
             :src="brand.picUrl"
             background-size="cover" />
        <div class="info-box">
          <div class="txt">{{brand.name}}</div>
          <div class="line"></div>
        </div>
      </div>
      <div class="desc">
        {{brand.desc}}
      </div>
    </div>
    <!-- 分类树列表 -->
<van-tree-select
  height="100%"
  :items="items"
  :main-active-index.sync="activeIndex"
>
  <template slot="content">
    <van-row gutter v-for="(item,index) in items" 
    :key=index
    v-if="activeIndex === index">
      <van-col span="24"
               v-for="(goods ,index) in brandGoods[item.text]"
               :key="index">
        <router-link :to="{ path: `/items/detail/${goods.id}`}">
          <img :src="goods.picUrl"
               style="width:150px;height:150px;">
        </router-link>
        <span style="margin-left: 30px;"><van-icon name="add" color="0066FF" size="30" @click="showSKU(goods.id)"/></span>
        <div style="margin-left: 20px; rgb(123, 116, 116);">{{goods.name}}</div>
        <div style="margin-left: 20px; color:#ab956d">￥ {{goods.retailPrice}}</div>
      </van-col>
    </van-row>
  </template>
</van-tree-select>
<van-sku
    close-on-click-overlay
      v-model="showSku"
      :sku="sku"
      :hide-stock="true"
      :goods="skuGoods"
      :goodsId="goods.info.id"
      @buy-clicked="addCart"
    >
    <template slot="sku-actions" slot-scope="props">
    <div class="van-sku-actions">
      <van-button
        square
        size="large"
        type="info"
        @click="props.skuEventBus.$emit('sku:buy')"
      >
        加入购物车
      </van-button>
    </div>
  </template>
    </van-sku>

<!-- 商品提交bar的价格会把小数点前移2位，暂时找不到其他解决办法这里手动乘以100 -->
    <van-submit-bar
    button-type="info"
      :disabled="cartInfo==0"
      :price="cartAllPrice*100"
      buttonText="去结算"
      label="总计"
      @submit="cartSubmit"
    >
    <span style="margin-left: 30px;"><van-icon @click="toCart" name="cart" color="0066FF" size="30" :info="(cartInfo > 0) ? cartInfo : ''"/></span>
    </van-submit-bar>

  </div>
</template>

<script>
import { goodsDetail,brandDetail, goodsList,cartGoodsCount,cartAdd } from '@/api/api';
import { Card, Row, Col,TreeSelect, SubmitBar,Sku   } from 'vant';
import _ from 'lodash';

export default {
  props: {
    brandId: [String, Number]
  },
  data() {
    return {
      goods: {
        userHasCollect: 0,
        info: {
          gallery: []
        }
      },
      sku: {
        tree: [],
        list: [],
        price: '1.00' // 默认价格（单位元）
      },
      skuGoods: {
        // 商品标题
        title: '',
        // 默认商品 sku 缩略图
        picture: ''
      },
      showSku: false,
      cartInfo: 0,
      cartAllPrice: 0,
      activeIndex: 0,
       items: [{ text: '' }],
      brand: {},
      brandGoods: []
    };
  },

  created() {
    this.init();
  },

  methods: {
    getProductIdByOne(s1) {
      var productId;
      var s1_name;
      _.each(this.goods.specificationList, specification => {
        _.each(specification.valueList, specValue => {
          if (specValue.id === s1) {
            s1_name = specValue.value;
            return;
          }
        });
      });

      _.each(this.goods.productList, v => {
        let result = _.without(v.specifications, s1_name);
        if (result.length === 0) {
          productId = v.id;
        }
      });
      return productId;
    },
    cartSubmit(){
      //还没想好怎么处理先暂时指向购物车页面
      this.$router.push({
        name: 'cart'
      });
    },
    addCart(data) {
      let that = this;
      let params = {
        goodsId: data.goodsId,
        number: data.selectedNum,
        productId: 0
      };
      if (_.has(data.selectedSkuComb, 's3')) {
        this.$toast({
          message: '目前仅支持两规格',
          duration: 1500
        });
        return;
      } else if (_.has(data.selectedSkuComb, 's2')) {
        params.productId = this.getProductId(
          data.selectedSkuComb.s1,
          data.selectedSkuComb.s2
        );
      } else {
        params.productId = this.getProductIdByOne(data.selectedSkuComb.s1);
      }
      cartAdd(params).then(() => {
        //重新请求购物车数据
        cartGoodsCount().then(res => {
        this.cartInfo = res.data.data;
         this.cartAllPrice = res.data.allprice;
      });
        this.$toast({
          message: '已添加至购物车',
          duration: 1500
        });
        that.showSku = false;
      });
    },
    init() {
       cartGoodsCount().then(res => {
        this.cartInfo = res.data.data;
         this.cartAllPrice = res.data.allprice;
      });
      brandDetail({
        id: this.brandId
      }).then(res => {
        this.brand = res.data.data;
      });

      goodsList({
        brandId: this.brandId
      }).then(res => {
        // 改为分类商品数据原来为this.brandGoods = res.data.data.list;
        this.brandGoods = res.data.data.categroyList;
        // 处理分类模块适应items的格式
        this.items=res.data.data.filterCategoryList.map((item)=>{return {text:item.name}});
      });
    },
    itemClick(id) {
      this.$router.push(`/items/detail/${id}`);
    },
    showSKU(id){
      goodsDetail({ id: id }).then(res => {
        this.goods = res.data.data;
        this.skuAdapter();
        this.showSku=true;
      });

    },
    toCart() {
      this.$router.push({
        name: 'cart'
      });
    },
    skuAdapter() {
      const tree = this.setSkuTree();
      const list = this.setSkuList();
      const skuInfo = {
        price: parseInt(this.goods.info.retailPrice), // 未选择规格时的价格
        stock_num: 0, // TODO 总库存
        collection_id: '', // 无规格商品skuId取collection_id，否则取所选sku组合对应的id
        none_sku: false, // 是否无规格商品
        hide_stock: true
      };
      this.sku = {
        tree,
        list,
        ...skuInfo
      };
      this.skuGoods = {
        title: this.goods.info.name,
        picture: this.goods.info.picUrl
      };
    },
    setSkuList() {
      var sku_list = [];
      _.each(this.goods.productList, v => {
        var sku_list_obj = {};
        _.each(v.specifications, (specificationName, index) => {
          sku_list_obj['s' + (~~index + 1)] = this.findSpecValueIdByName(
            specificationName
          );
        });

        sku_list_obj.price = v.price * 100;
        sku_list_obj.stock_num = v.number;
        sku_list.push(sku_list_obj);
      });

      return sku_list;
    },
    findSpecValueIdByName(name) {
      let id = 0;
      _.each(this.goods.specificationList, specification => {
        _.each(specification.valueList, specValue => {
          if (specValue.value === name) {
            id = specValue.id;
            return;
          }
        });
        if (id !== 0) {
          return;
        }
      });
      return id;
    },
    setSkuTree() {
      let that = this;
      let specifications = [];
      _.each(this.goods.specificationList, (v, k) => {
        let values = [];
        _.each(v.valueList, vv => {
          vv.name = vv.value;
          values.push({
            id: vv.id,
            name: vv.value,
            imUrl: vv.picUrl
          });
        });

        specifications.push({
          k: v.name,
          v: values,
          k_s: 's' + (~~k + 1)
        });
      });

      return specifications;
    },
  },

  components: {
    [Sku.name]: Sku ,
    [SubmitBar.name]: SubmitBar,
    [TreeSelect .name]: TreeSelect ,
    [Card.name]: Card,
    [Row.name]: Row,
    [Col.name]: Col
  }
};
</script>

<style lang="scss" scoped>
.goods_brand {
  .brand-info {
    .name {
      width: 100%;
      height: 180px;
      position: relative;

      .img {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 180px;
      }

      .info-box {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 180px;
        text-align: center;
        display: flex;
        justify-content: center;
        align-items: center;
        display: block;

        .txt {
          display: block;
          margin-top: 60px;
          height: 35px;
          font-size: 35px;
          color: #fff;
        }

        .line {
          margin: 0 auto;
          margin-top: 16px;
          display: block;
          height: 2px;
          width: 145px;
          background: #fff;
        }
      }
    }
    .desc {
      background: #fff;
      width: 100%;
      height: auto;
      overflow: hidden;
      padding: 25px 20px;
      font-size: 20px;
      color: #666;
      line-height: 20px;
      text-align: center;
    }
  }
}
</style>