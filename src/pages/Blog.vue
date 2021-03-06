<template>
  <div class="page-wrapper editor-wrapper">
    <div class="page-header">
      <div class="common-btn back-btn" @click="$router.go(-1)">返回</div>
      <div class="page-header-content">
        <edit-header v-if="isAuthor" :showDelete="type === '2' && isAuthor"  @add-img="addImg" @clear-html="clearOpt" @deltet-blog="deleteOpt" @add-blog="submitOpt"></edit-header>
        <div v-else class="page-header-title">{{blogData.user && blogData.user.nickname}}的博客</div>
      </div>
    </div>
    <div class="page-body">
      <div class="page-body-content">
        <userCommon v-if="type === '1'" :userInfo="userInfo">
          <template slot="user-others">
            <div class="info-item">作者</div>
          </template>
        </userCommon>
        <userCommon v-else :userInfo="blogData.user">
          <template slot="user-others">
            <div class="info-item">作者</div>
          </template>
        </userCommon>
        <div id="content-wrapper" class="content-wrapper" :contenteditable="isAuthor" v-html="blogData.content"></div>
        <div class="blog-info" v-if="type === '2'">
          <div class="blog-options">
            <div class="blog-time">{{blogData.type === '1' ? `${formatTime(blogData.createdAt)}发布` : `${formatTime(blogData.updatedAt)}更新`}}</div>
            <div class="blog-view">浏览 {{blogData.view || 0}} 次</div>
            <div class="favor-btn common-btn" @click="likeOpt">{{likeData.isLiked ? '已' : ''}}喜欢</div>
            <div v-if="!userInfo || !isAuthor" class="favor-btn common-btn" @click="collectOpt">{{collectData.isCollected ? '已' : ''}}收藏</div>
          </div>
          <div class="blog-likes blog-common" v-if="likeData.list.length > 0">
            <div class="title">喜欢（{{likeData.list.length}}人）：</div>
            <div class="likes-conent">
              <div class="likes-item" v-for="(item, index) in likeData.list" :key="index">
                <div class="likes-info">
                  <userCommon :userInfo="item.user" class="small-avatar"></userCommon>
                </div>
              </div>
            </div>
          </div>
          <div class="blog-likes blog-common" v-if="collectData.list.length > 0">
            <div class="title">收藏（{{collectData.list.length}}人）：</div>
            <div class="likes-conent">
              <div class="likes-item" v-for="(item, index) in collectData.list" :key="index">
                <div class="likes-info">
                  <userCommon :userInfo="item.user" class="small-avatar"></userCommon>
                </div>
              </div>
            </div>
          </div>
          <div class="blog-comments blog-common">
            <div class="title">评论（{{commentData.total}}条）：</div>
            <div class="comments-content">
              <textarea cols="30" rows="3" placeholder="不超过100字" class="comments-input" v-model="commentContent"></textarea>
              <div class="comments-options">
                <div class="comments-tips"></div>
                <div class="comments-btn common-btn" @click="commentOpt">提交评论</div>
              </div>
              <div class="comments-list">
                <div class="comments-item" v-for="(item, index) in commentData.list" :key="index">
                  <div class="comments-user">
                    <userCommon :userInfo="item.user" class="small-avatar"></userCommon>
                    <div class="comments-time">{{formatTime(item.createdAt)}}</div>
                  </div>
                  <div class="comments-detail">{{item.comment}}</div>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="page-tip">
          <span v-if="commentData.isEnding">没有更多评论啦~</span>
          <span v-else-if="!commentData.isEnding && isLoading && commentData.total >= 15">加载中~</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { formatTime } from '../plugins/func'

import editHeader from '../components/editHeader.vue'
import userCommon from '../components/userCommon.vue'

import { createNamespacedHelpers } from 'vuex'
const { mapGetters, mapActions } = createNamespacedHelpers('blog')

export default {
  data () {
    return {
      contentWrapper: null,
      type: '1',
      commentContent: '',
      isLoading: false
    }
  },
  components: {
    editHeader,
    userCommon
  },
  async created () {
    if (this.$route.query.blogid) {
      this.type = '2'
    } else {
      this.type = '1'
    }
    await this.getBlog({type: this.type, blogid: this.$route.query.blogid || ''})
    if (this.type === '2' && (!this.userInfo || (this.blogData.user && this.blogData.user.userId !== this.userInfo.userId))) {
      this.viewBlog()
    }
  },
  mounted () {
    this.contentWrapper = document.getElementById('content-wrapper')
    if (!this.$route.query.blogid) {
      this.contentWrapper.focus()
    }
    let that = this
    this.$nextTick(() => {
      document.querySelector('.editor-wrapper').addEventListener('scroll', async (e) => {
        if (that.type === '2' && e.target.offsetHeight + e.target.scrollTop + 100 >= e.target.scrollHeight && !that.isLoading && !that.commentData.isEnding) {
          that.isLoading = true
          await that.viewCommons({pageDown: true})
          that.isLoading = false
        }
      })
    })
  },
  computed: {
    ...mapGetters(['userInfo', 'blogData', 'isAuthor', 'likeData', 'collectData', 'commentData'])
  },
  methods: {
    ...mapActions(['getBlog', 'createBlog', 'updateBlog', 'deleteBlog', 'likeBlog', 'unlikeBlog', 'viewBlog', 'viewCommons', 'commentBlog', 'uncollectBlog', 'collectBlog']),
    formatTime,
    insertHtmlAtCaret (str) {
      let sel, range
      if (window.getSelection) {
        sel = window.getSelection()
        if (sel.getRangeAt && sel.rangeCount) {
          range = sel.getRangeAt(0)
          range.deleteContents()
          let el = document.createElement('div')
          el.innerHTML = str
          let frag = document.createDocumentFragment()
          let node
          let lastNode
          while ((node = el.firstChild)) {
            lastNode = frag.appendChild(node)
          }
          range.insertNode(frag)
          if (lastNode) {
            range = range.cloneRange()
            range.setStartAfter(lastNode)
            range.collapse(true)
            sel.removeAllRanges()
            sel.addRange(range)
          }
        }
      } else if (document.selection && document.selection.type !== 'Control') {
        document.selection.createRange().pasteHTML(str)
      }
    },
    getBlogInfo (blog) {
      let picArr = []
      let imgReg = /<img.*?(?:>|\/>)/gi
      let srcReg = /src=['"]?([^'"]*)['"]?/i
      let htmlReg = /<[^>]+>/gim
      let paragraph = blog.replace(htmlReg, '')
      if (paragraph.length > 150) {
        paragraph = paragraph.substr(0, 150) + '...<span class="blog-more">查看更多</span>'
      }
      let arr = blog.match(imgReg)
      if (arr) {
        arr.forEach(element => {
          let src = element.match(srcReg)
          if (src[1]) {
            picArr.push(src[1])
          }
        })
        return {
          cover: picArr[0],
          paragraph
        }
      } else {
        return {
          cover: '',
          paragraph
        }
      }
    },
    addImg (url) {
      this.contentWrapper.focus()
      let flg = '<img src="' + url + '">'
      this.insertHtmlAtCaret(flg)
    },
    clearOpt () {
      this.contentWrapper.innerHTML = ''
      this.contentWrapper.focus()
    },
    async deleteOpt () {
      let res = await this.deleteBlog()
      if (res) {
        this.$toast({
          title: '删除成功'
        })
        this.$router.go(-1)
      }
    },
    async submitOpt () {
      if (this.contentWrapper.innerHTML.replace(/(^\s*)|(\s*$)/, '') === '') {
        return
      }
      let blog = this.contentWrapper.innerHTML.replace(/(^\s*)|(\s*$)/, '').replace(/<script[^>]*>[\s\S]*?<\/[^>]*script>/gi, '')
      let blogInfo = this.getBlogInfo(blog)
      let params = {
        params1: {
          cover: blogInfo.cover,
          paragraph: blogInfo.paragraph
        },
        params2: {
          content: blog
        }
      }
      if (this.type === '1') {
        let res = await this.createBlog(params)
        if (res) {
          this.$toast({
            title: '发布成功'
          })
          this.$router.go(-1)
        }
      } else if (this.type === '2' && this.$route.query.blogid && this.userInfo.objectId === this.blogData.userid) {
        let res = await this.updateBlog(params)
        if (res) {
          this.$toast({
            title: '更新成功'
          })
          this.$router.go(-1)
        }
      }
    },
    likeOpt () {
      if (!this.userInfo) {
        this.$toast({
          title: '请先登录'
        })
        return
      }
      if (this.likeData.isLiked) {
        this.unlikeBlog()
      } else {
        this.likeBlog()
      }
    },
    collectOpt () {
      if (!this.userInfo) {
        this.$toast({
          title: '请先登录'
        })
        return
      }
      if (this.collectData.isCollected) {
        this.uncollectBlog()
      } else {
        this.collectBlog()
      }
    },
    async commentOpt () {
      if (!this.userInfo) {
        this.$toast({
          title: '请先登录'
        })
        return
      }
      let comment = this.commentContent.replace(/(^\s*)|(\s*$)/, '')
      if (comment === '') {
        return
      }
      if (comment.length > 100) {
        this.$toast({
          title: '评论不超过100字'
        })
        return
      }
      let res = await this.commentBlog({comment})
      if (res) {
        this.commentContent = ''
      }
    }
  }
}
</script>

<style lang="less" scoped>
@import '../assets/css/color.less';
.editor-wrapper{
  .page-body-content{
    .content-wrapper{
      border: none;
      outline: none;
      padding: 20px 40px;
      line-height: 18px;
    }
    .blog-info{
      padding: 0 40px;
      .blog-options{
        display: flex;
        align-items: center;
        font-size: 11px;
        .blog-time{
          flex: 1;
          color: @gray;
        }
        .blog-view{
          color: @gray;
          margin-right: 15px;
        }
        .favor-btn{
          font-size: 11px;
        }
      }
      .blog-common{
        margin-top: 30px;
        .title{
          font-size: 13px;
          font-weight: bold;
        }
      }
      .blog-likes{
        .likes-conent{
          font-size: 11px;
          padding-left: 15px;
          .likes-item{
            display: inline-block;
            height: 25px;
            margin-top: 10px;
            margin-left: 15px;
            &:first-child{
              margin-left: 0;
            }
            .likes-info{
              display: flex;
              align-items: center;
            }
          }
        }
      }
      .blog-comments{
        .comments-content{
          font-size: 12px;
          padding-left: 15px;
          .comments-input{
            display: block;
            width: 100%;
            margin-top: 15px;
            resize: none;
            font-size: 12px;
            padding: 5px 8px;
          }
          .comments-options{
            display: flex;
            align-items: center;
            margin-top: 10px;
            .comments-tips{
              flex: 1;
            }
            .comments-btn{
              font-size: 11px;
            }
          }
          .comments-list{
            margin-top: 15px;
            .comments-item{
              margin-top: 20px;
              .comments-user{
                display: flex;
                align-items: center;
                margin-right: 10px;
                .comments-time{
                  font-size: 11px;
                  flex: 1;
                  text-align: right;
                  color: @gray;
                }
              }
              .comments-detail{
                padding-left: 30px;
                color: @gray;
              }
            }
          }
        }
      }
    }
    .page-tip{
      margin-top: 15px;
    }
  }
}
</style>
