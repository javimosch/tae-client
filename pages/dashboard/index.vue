<template>
<div class="Dashboard pt-5 pb-5 container">
  <div class="row no-gutters">
    <div class="col-12 mb-5" v-show="!isLogged">
      <TaeCreateAccountForm></TaeCreateAccountForm>
    </div>
    <div class="col-12">
      <h4 class="text-center mb-4">How it works: Select a project, copy paste the script "et voilà"</h4>
    </div>
    <div class="col-12">
      
      <TaeProjectCardSelect class="mb-2 mt-2" v-model="project._id"></TaeProjectCardSelect>
      

      
    </div>
    <div class="col-12">
      <TaeErrorsTable @select="viewError"
                      :items="items"></TaeErrorsTable>
    </div>
    <div class="col-12 mb-5" v-show="!isLogged">
      <TaeCreateAccountForm></TaeCreateAccountForm>
    </div>
    
  </div>
  <ErrorModalDetails :item.sync="item"></ErrorModalDetails>
</div>



</template>

<script>
import ErrorModalDetails from '@/components/ErrorModalDetails';
import TaeCreateAccountForm from '@/components/TaeCreateAccountForm';
import TaeProjectCardSelect from '@/components/controls/TaeProjectCardSelect';
import CollapsableCard from '@/components/CollapsableCard';
import CopyPasteScript from '@/components/CopyPasteScript';
import TaeErrorsTable from '@/components/TaeErrorsTable';
import { call } from '@/plugins/rpcApi';
import { watchProjectSocket } from '@/plugins/taeSockets';
import moment from 'moment';
export default {
  layout: 'protected',
  name: 'Dashboard',
  props: [],
  fetch() {},
  data() {
    return {
      projects: [],
      project: {
        _id: ''
      },
      item: {
        _id: null
      },
      items: [],
      watcher: null
    }
  },
  async asyncData() {
    return {}
  },
  computed: {
    isLogged(){
      return this.$store.state.auth.isLogged
    }
  },
  methods: {
    viewError(err) {
      this.item = err
      console.warn(err);
    }
  },
  components: {
    TaeErrorsTable,
    CopyPasteScript,
    CollapsableCard,
    TaeProjectCardSelect,
    TaeCreateAccountForm,
    ErrorModalDetails
  },
  created() {},
  destroyed() {
    if(typeof this.watcher ==='function') this.watcher();
  },
  async mounted() {
    const self = this

    if (process.server) {
      return
    }

    this.$watch('project._id', async(v) => {
      if(!v)return;
      self.items = (await call('findPaginate', {
        model: 'tae_error',
        query: {
          project:v,
          updatedAt: {
            $gte: moment().subtract('1', 'month')._d
          }
        },
        options: {
          sort: '-updatedAt'
        }
      })).map(o=>{
        o.count = o.count.toString()
        return o;
      })


      if(typeof self.watcher==='function'){
        self.watcher();//close socket
      }
      self.watcher = watchProjectSocket(v, {
        onError: err => {
          err.count = err.count.toString()
          let record = this.items.find(d => d._id === err._id)
          if (record) {
            // console.log('Error update',err.message);
            Object.assign(record, err)
          } else {
            // console.log('Error added',err.message);
            self.items.push(err)
          }
          self.items = _.orderBy(self.items, [
            'lastAt'
          ], [
            'desc'
          ])
        }
      })
    })

    await this.$store.dispatch('auth/update')
    await this.$store.dispatch('auth/ensureSession')
    
    this.projects = await call('taeGetPersonalProjects')
    Object.assign(this.project,this.projects[0])
  }
}

</script>

<style lang="scss" scoped="">
.Dashboard {
  background-color: $color3;
}
h4{
  color:$color2;
}
</style>
