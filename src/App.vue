<template>
  <div id="app" >
    <article class="media" v-for="comment in comments">
      <figure class="media-left">
        <p class="image is-48x48">
          <img :src="comment.avatar">
        </p>
      </figure>
      <div class="media-content">
        <strong>{{comment.user}}</strong> <small>commented on</small> <small>{{comment.updated_at}}</small>
        <div class="content" v-html="comment.content"></div>
      </div>
    </article>
    <nav class="media pagination is-small" v-if="pageCount>1" role="navigation" aria-label="pagination">
      <a class="pagination-previous" v-if="pageIndex>1" @click="getComments(pageIndex-1)">Prev</a>
      <a class="pagination-previous" v-else disabled>Prev</a>
      <a class="pagination-next" v-if="pageIndex<pageCount" @click="getComments(pageIndex+1)">Next</a>
      <a class="pagination-next" v-else disabled>Next</a>
      <ul class="pagination-list">
        <li v-for="page in pages">
          <span class="pagination-ellipsis" v-if="page.ellipsis">&hellip;</span>
          <a class="pagination-link button is-current" v-else-if="page.selected" v-bind:class="{'is-loading': loading}" @click="getComments(page.index)">{{page.content}}</a>
          <a class="pagination-link" v-else @click="getComments(page.index)">{{page.content}}</a>
        </li>
      </ul>
    </nav>
    <article class="media" v-if="token">
      <figure class="media-left">
        <p class="image is-64x64">
          <img :src="avatar">
        </p>
      </figure>
      <div class="media-content">
        <div class="field">
          <p class="control">
            <textarea class="textarea" v-model="comment" placeholder="Add a comment..."></textarea>
          </p>
        </div>
        <a class="button is-info" v-on:click="createComment">Submit</a>
      </div>
    </article>
    <div class="media level" v-if="!token">
        <div class="media-content level-item">
          <a class="button is-info" :href="authUrl">Login to Comment</a>
        </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import md5 from 'md5'
const API_URL = "https://api.github.com"
const AUTH_URL = "https://github.com/login/oauth/authorize"
const DEFAULT_SERVER = "http://gocomment.mikesu.net"
const DEFAULT_CLIENT_ID = "beaff20a3a55ee7c9eb5"
const SCOPE = 'public_repo'
const PRE_PAGE = 10
const TOKEN_KEY = 'go_comment_github_token'
export default {
  name: 'app',
  props: ['owner','repo','pid','title','client_id','server_url'],
  data () {
    console.log("data")
    return {
      token: getToken(),
      avatar:'',
      comment:'',
      pageCount: 0,
      pageIndex: 1,
      comments : [],
      loading: true
    }
  },
  computed: {
    authUrl (){
      var authUrl = AUTH_URL+ "?client_id=" + this.client_id
      authUrl = authUrl + "&scope=" + SCOPE
      authUrl = authUrl + "&redirect_uri=" + this.server_url
      authUrl = authUrl + "?url=" + window.btoa(getUrl())
      return authUrl
    },
    pages:pages 
  },
  created () {
    console.log("created")
    this.server_url = this.server_url || DEFAULT_SERVER
    this.client_id = this.client_id || DEFAULT_CLIENT_ID
    this.loadUser()
    this.loadComments()
  },
  methods:{
    loadUser:loadUser,
    loadComments:loadComments,
    listComments:listComments,
    createComment:createComment
  }
}

function loadUser() {
  console.log(this.token)
  if(this.token){
    var userApiPath = API_URL + '/user'
    var option = {
      headers:{
        'Authorization':'token '+this.token
      } 
    }
    axios.get(userApiPath,option).then((response) => {
      console.log(response)
      this.avatar = response.data.avatar_url
      this.user = response.data.login
    }).catch((error) => {
      console.log(error)
      this.token = 0
      cleanToken()
    })
  }
}

function loadComments() {
  var q = md5(this.pid) + ' in:body repo:' + this.owner + '/' + this.repo + ' type:issue'
  var issueApiPath = API_URL + '/search/issues?q=' + encodeURIComponent(q)
  axios.get(issueApiPath).then((response) => {
    console.log(response)
    if(response.data.items.length>0){
      var data = response.data.items[0]
      this.comments_url = data.comments_url
      this.pageCount = Math.ceil(data.comments/PRE_PAGE)
      this.listComments(1)
    }
  })
}

function listComments(pageIndex){
  this.loading = true
  this.pageIndex = pageIndex
  var option = {
    headers:{
      'Accept':'application/vnd.github.html+json'
    },
    params:{
      page:pageIndex,
      per_page:PRE_PAGE
    }
  }
  axios.get(this.comments_url,option).then((response) =>{
    console.log(response)
    this.comments = []
    for (var i=0;i<response.data.length;i++){
      var data = response.data[i]
      var comment = { 
        user:data.user.login,
        avatar:data.user.avatar_url,
        content: data.body_html,
        created_at: new Date(data.created_at).toDateString(),
        updated_at: new Date(data.updated_at).toDateString()
      }
      this.comments.push(comment)
    }
    this.loading = false
  })
}

function createComment() {
  var option ={
    headers:{
      'Content-Type':'application/json',
      'Authorization':'token '+this.token
    }
  }
  if(this.comments_url){
    var data = {body: this.comment}
    console.log(data)
    axios.post(this.comments_url,data,option).then((response) => {
      console.log(response)
      this.listComments(1)
    }).catch((error) => {
      console.log(error)
    })
  }else{
    var data = {
      title: this.title,
      body: getUrl()+'\n`' + md5(this.pid) + '`'
    }
    console.log(data)
    var issueApiPath = API_URL + '/repos/' + this.owner + '/' + this.repo  + '/issues'
    axios.post(issueApiPath,data,option).then((response) =>{
      this.comments_url = response.data.comments_url
      this.createComment()
    }).catch((error) => {
      console.log(error)
    })
  }
}

function pages() {
  let items = {}
  if (this.pageCount<8) {
    for(let index = 1; index <= this.pageCount; index++){
      let page = {
        index: index,
        content: index,
        ellipsis: false,
        selected: index === this.pageIndex
      }
      items[index] = page
    }
  }else{
    let pageIndex = 1
    let ellipsis = false
    for(let index = 1; index < 8 ; index++){
      ellipsis = false
      if (index==2&&this.pageIndex>4) {
        ellipsis = true
        if (this.pageCount-this.pageIndex>3) {
          pageIndex = this.pageIndex - 2 
        }else{
          pageIndex = this.pageCount - 5 
        }
      }
      if (index==6&&this.pageCount-this.pageIndex>3) {
        ellipsis = true
        pageIndex = this.pageCount -1 
      }

      let page = {
        index: pageIndex,
        content: pageIndex,
        ellipsis: ellipsis,
        selected: pageIndex === this.pageIndex
      }
      pageIndex++
      items[index] = page
    }
  }
  return items
}

//static function
function getToken() {
  // get token in URL
  var query = queryParse()
  var token = query['access_token']
  if(token){
    window.localStorage.setItem(TOKEN_KEY,token)
  }else{
    token = window.localStorage.getItem(TOKEN_KEY)
  }
  return token
}

function cleanToken(){
  window.localStorage.removeItem(TOKEN_KEY)
}

function getUrl() {
  var query = queryParse()
  var originQuery = {}
  Object.keys(query).forEach(key => {
    if (key!='access_token'&&key!='scope'&&key!='token_type'){
      originQuery[key] = query[key]
    } 
  })
  var url = window.location.protocol + "//"
  url = url + window.location.host
  url = url + window.location.pathname
  url = url + queryStringify(originQuery)
  return url
}

function queryParse(search = window.location.search) {
    if (!search) return {}
    const queryString = search[0] === '?' ? search.substring(1) : search
    const query = {}
    queryString.split('&')
      .forEach(queryStr => {
        const [key, value] = queryStr.split('=')
        if (key) query[key] = value
      })

    return query
}

function queryStringify(query, prefix = '?') {
    const queryString = Object.keys(query)
      .map(key => `${key}=${encodeURIComponent(query[key] || '')}`)
      .join('&')
    return queryString ? prefix + queryString : ''
}
</script>

<style lang="scss">
@import "~bulma/bulma";
</style>