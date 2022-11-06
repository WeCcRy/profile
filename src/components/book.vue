<template>
  <a-layout>
    <a-layout-head style="padding: 16px 10px 0 16px">
      <a-row type="flex" justify="space-around" align="middle">
      <!--      style="text-align: left;font-weight: bold;margin-left: 15px"-->
      <a-col :span="2"><a-typography-title :level="2">{{year}}<SmileTwoTone twoToneColor="#002FA7" style="margin-left: 15px"/></a-typography-title></a-col>
      <a-col :span="1" style="margin-bottom: 15px">
        <a-dropdown>
          <template #overlay>
            <a-menu @click="handleMenuClick">
              <a-menu-item key="2021" v-if="year=='2021'?0:1">2021</a-menu-item>
              <a-menu-item key="2022" v-if="year=='2022'?0:1">2022</a-menu-item>
            </a-menu>
          </template>
          <a-button>
            年份选择
            <DownOutlined />
          </a-button>
        </a-dropdown>
      </a-col>
      <a-col :span="18"></a-col>
      <a-col :span="2"><a-button type="link" @click="showDrawer">我的图书报告</a-button></a-col>
      <a-col :span="1">
        <a-button v-if="year==2022" type="primary" size="small" @click="showModal">
          <template #icon><plus-outlined /></template>
        </a-button>
      </a-col>
    </a-row>
    </a-layout-head>
    <a-layout-content style="margin: 0 16px">
      <a-table :columns="columns" :data-source="bookList" :pagination="false" rowKey="key">
        <template #bodyCell="{ column, record }">
          <template v-if="column.key === 'name'">
            <a>
              <a-popover :title="record.name" placement="bottom" @mouseenter="getDetails(record.ISBN)">
                <template #content >
                  <p v-if="isLoading">加载中...</p>
                  <img :src="'https://images.weserv.nl/?url='+photoUrl" style="width: 81px;height: 117px;float: left" alt="图片加载中...">
                  <div style="float: left;width: 250px;padding-left: 10px" >{{bookDetail}}</div>
                  <div style="clear:both"></div>
                </template>
                {{ record.name }}
              </a-popover>
            </a>
          </template>
          <template v-else-if="column.key === 'tags'">
        <span>
          <a-tag
              v-for="tag in record.tags"
              :key="tag"
              :color=color(tag)
          >
            {{tag}}
          </a-tag>
        </span>
          </template>
          <!--      <template v-else-if="column.key === 'action'">-->
          <!--        <span>-->
          <!--          <a>Invite 一 {{ record.name }}</a>-->
          <!--          <a-divider type="vertical" />-->
          <!--          <a>Delete</a>-->
          <!--          <a-divider type="vertical" />-->
          <!--          <a class="ant-dropdown-link">-->
          <!--            More actions-->
          <!--            <down-outlined />-->
          <!--          </a>-->
          <!--        </span>-->
          <!--      </template>-->
        </template>
      </a-table>
    </a-layout-content>
  <a-drawer width="640" placement="right" :closable="false" :visible="drawerVisible" @close="onClose">
    <p :style="[pStyle, pStyle2]">我的{{year}}</p>
    <p :style="pStyle">Personal</p>
    <a-row>
      <a-col :span="12">
      </a-col>
      <a-col :span="12">
      </a-col>
    </a-row>
  </a-drawer>
    <a-modal v-model:visible="formVisible" title="添加图书" @ok="handleOk">
      <a-form :model="addBookInfo" @finish="onFinish">
        <a-form-item name="ISBN" label="ISBN" has-feedback required :rules="[{type: 'string', min: 13, max: 13 ,message:'请输入13位ISBN码'}]">
          <a-input v-model:value="addBookInfo.ISBN" />
        </a-form-item>
        <a-form-item name="isFiction" label="小说(是/否)">
          <a-switch v-model:checked="addBookInfo.isFiction" />
        </a-form-item>
        <a-form-item name="type" label="标签">
          <a-checkbox-group v-model:value="addBookInfo.type">
            <a-checkbox value="推理" name="type">推理</a-checkbox>
            <a-checkbox value="历史" name="type">历史</a-checkbox>
            <a-checkbox value="纪实" name="type">纪实</a-checkbox>
            <a-checkbox value="社会" name="type">社会</a-checkbox>
            <a-checkbox value="科幻" name="type">科幻</a-checkbox>
            <a-checkbox value="哲学" name="type">哲学</a-checkbox>
          </a-checkbox-group>
        </a-form-item>
        <a-form-item name="bookRate" label="评分">
          <a-input-number v-model:value="addBookInfo.bookRate" :min="0" :max="5" :precision="2" :step="0.1"/>
          <a-rate :value="bookRateVShow" allow-half disabled style="margin-left: 20px"/>
        </a-form-item>
        <a-form-item :wrapper-col="{ span: 14, offset: 7 }">
          <a-button @click="bookClear">清空</a-button>
          <a-button type="primary" html-type="submit" style="margin-left: 10px">查看详细信息</a-button>
        </a-form-item>
      </a-form>
      <a-card style="width: 190px;margin:auto" v-if="isChecked" >
        <template #cover>
          <img :src="bookAddDetail.photoUrl" style="width: 190px;height: 263px" alt="加载中..."/>
        </template>
        <a-card-meta :title="bookAddDetail.title">
          <template #description>{{ bookAddDetail.Author }}</template>
        </a-card-meta>
      </a-card>
      <template #footer>
        <a-button key="back" @click="handleCancel">取消</a-button>
        <a-button key="submit" type="primary" @click="handleOk" :disabled="!isChecked">{{bookAddConfirm}}</a-button>
      </template>
    </a-modal>
  </a-layout>
</template>
<script>

import { DownOutlined,SmileTwoTone,PlusOutlined } from '@ant-design/icons-vue';
import { message } from 'ant-design-vue';
import {defineComponent, ref, watch, reactive, toRaw} from 'vue';
import axios from 'axios'
const columns = [
  {
    title:"排名",
    dataIndex: "index",
    key:"index",
    customRender:(text)=> {
      return parseInt(text.index)+1
    }
  },
  {
    title:'书名',
    name: 'Name',
    dataIndex: 'name',
    key: 'name',
  },
  {
    title: '作者',
    dataIndex: 'editor',
    key: 'editor',
  },
  {
    title: '标签',
    key: 'tags',
    dataIndex: 'tags',
  },
  {
    title: '评分',
    key: 'rate',
    dataIndex:'rate',
    customRender:(text)=> {
      const rate=parseFloat(text.value)
      let x=parseInt(rate)
      let string=""
      for(let i=0;i<x;i++){
        string+="🌕"
      }
      let y=rate%1
      if(y==0.5){
        string+="🌗"
      }else if(y<0.5&&y>0){
        string+="🌘"
      }else if(y>0.5){
        string+="🌖"
      }
      if(string.length<10){
        for(let i=string.length;i<10;i+=2){
          string+="🌑"
        }
      }
      return (string)
    },
    defaultSortOrder: 'descend',
    sorter: (a, b) => a.rate - b.rate,
  },
];
export default defineComponent({
  components: {
    DownOutlined,
    SmileTwoTone,
    PlusOutlined,
  },
  setup() {
    let year=ref("2022")
    let bookList=ref([])
    let bookDetail=ref("")
    let isLoading=ref(true)
    let photoUrl=ref("")
    //https://jike.xyz/jiekou/isbn.html#%E8%BF%94%E5%9B%9E%E5%8F%82%E6%95%B0%E8%AF%B4%E6%98%8E
    //私人接口APIKEY，如无法获得数据，需手动更改
    const apikey="13294.206eaa07ed2ea8b374cf076241a2024c.f57bd57f2303f9557c040b0365744e1c"
    const drawerVisible = ref(false)
    const formVisible = ref(false)
    let ISBNArray=[]
    let isChecked=ref(false)
    let bookRateVShow=ref(3)
    let bookAddConfirm=ref("请先查看书本详细信息")
    const pStyle = {
      fontSize: '16px',
      color: 'rgba(0,0,0,0.85)',
      lineHeight: '24px',
      display: 'block',
      marginBottom: '16px',
    };
    const pStyle2 = {
      marginBottom: '24px',
    };
    let addBookInfo = reactive({
      ISBN: '',
      isFiction: true,
      type: [],
      bookRate: 3,
    });
    let bookAddDetail=reactive({
      ISBN:'',
      title:'',
      author:'',
      photoUrl:'',
      description:'',
    })
    const onSubmit = () => {
      console.log('submit!', addBookInfo);
    };
    //通用清空方法
    const objClear=(obj)=>{
      Object.keys(obj).map(key=>{
        delete obj[key]
      })
    }
    const showDrawer = () => {
      drawerVisible.value = true;
    };
    const onClose = () => {
      drawerVisible.value = false;
    };
    const success = () => {
      message.success('This is a success message');
    };
    const handleMenuClick=(e)=>{
      year.value=e.key
    }
    const showModal = () => {
      formVisible.value = true;
    };
    const bookClear=()=>{
      addBookInfo.ISBN=''
      addBookInfo.bookRate=0
      addBookInfo.isFiction=true
      addBookInfo.type=[]
      isChecked.value=false
      objClear(bookAddDetail)
    }
    const handleCancel = () => {
      formVisible.value = false;
    };
    const handleOk = () => {
      console.log(toRaw(addBookInfo));
      let flag=0
      ISBNArray.forEach((ele)=>{
        if(ele==addBookInfo.ISBN&&flag==0){
          flag=1
        }
      })
      if(flag==1){
        message.error("该ISBN已存在")
      }
      else{
        axios.post('http://127.0.0.1/addBook',{
          title:bookAddDetail.title,
          author:bookAddDetail.author,
          photoUrl:bookAddDetail.photoUrl,
          description:bookAddDetail.description,
          ISBN:addBookInfo.ISBN,
          bookRate:addBookInfo.bookRate,
          isFiction:addBookInfo.isFiction==1?"小说":"非小说",
          type:addBookInfo.type
        })
            .then((res)=>{
              if(res.status==200){
                message.success("添加成功")
                formVisible.value=false
                getBookList(year)
              }
            }).catch(err=>{
          console.log(err);
        })
      }
    };
    const onFinish = values => {
      isChecked.value=true
      getDetailsAdd(values.ISBN)
    };
    const getBookList=year=>{
      axios.get(`http://127.0.0.1/booklist?year=${year.value}`)
          .then(res=>{
            bookList.value=res.data.data
            ISBNArray=res.data.ISBN
          }).catch(err=>{
        console.log(err);
      })
    }
    function getDetails(ISBN){
      isLoading.value=true
      bookDetail.value=""
      photoUrl.value=""
      axios.get(`https://api.jike.xyz/situ/book/isbn/${ISBN}?apikey=${apikey}`)
          .then(res=>{
            let data=res.data.data
            isLoading.value=false
            if(data.description.length>100) {
              bookDetail.value = data.description.slice(0,100)+"..."
            }
            photoUrl.value=data.photoUrl
          }).catch(err=>{
        console.log(err);
      })
    }
    function getDetailsAdd(ISBN){
      axios.get(`https://api.jike.xyz/situ/book/isbn/${ISBN}?apikey=${apikey}`)
          .then(res=>{
            console.log(res.data.data)
            let data=res.data.data
            bookAddDetail.photoUrl='https://images.weserv.nl/?url='+data.photoUrl
            bookAddDetail.author= data.author
            bookAddDetail.title=data.name
            bookAddDetail.description=data.description
          }).catch(err=>{
        console.log(err);
      })
    }
    const color=(name)=>{
      switch (name){
        case "小说":
          return 'blue'
        case "推理":
          return 'purple'
        case "历史":
          return 'gold'
        case "纪实":
          return 'orange'
        case "非纪实":
          return 'cyan'
        case "社会":
          return 'lime'
        case "科幻":
          return 'red'
        case "哲学":
          return 'pink'
        case "非小说":
          return 'green'
        default:
          return 'default'

      }
    }
    watch(year,(n,o)=>{
      getBookList(year)
      message.success("已为你切换至"+n+"年")
    })
    watch(isChecked,()=>{
      if(isChecked.value==false){
        bookAddConfirm.value="请先查看书本详细信息"
      }else{
        bookAddConfirm.value="提交"
      }
    })
    watch(() => addBookInfo.bookRate,()=>{
      let intRate=parseInt(addBookInfo.bookRate)
      if(addBookInfo.bookRate%1>=0.5){
        bookRateVShow.value=intRate+0.5
      }else{
        bookRateVShow.value=intRate
      }
    })
    getBookList(year)
    return {
      handleMenuClick,
      getDetails,
      bookList,
      drawerVisible,
      formVisible,
      pStyle,
      pStyle2,
      addBookInfo,
      isChecked,
      bookRateVShow,
      bookAddConfirm,
      bookAddDetail,
      bookClear,
      showDrawer,
      onClose,
      success,
      showModal,
      handleCancel,
      handleOk,
      getBookList,
      onSubmit,
      onFinish,
      isLoading,
      bookDetail,
      photoUrl,
      year,
      columns,
      color,
    };
  },
});
</script>
