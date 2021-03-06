<script>
import searchNode from './search-node.vue'
import { computSortNum, getSortData, getDictionary, deepCopy } from './utils.js'
export default {
  name: 'search-tree',
  components: { searchNode },
  props: {
    data: {                // 源数据
      type: Array,
      required: true
    },
    nodeKey: {             // 指定的id值
      type: String,
      default: 'id'
    },
    search: {              // 模糊搜索关键词
      type: String,
      default: ''
    },
    emptyText: {           // 内容为空时展示的文本
      type: String,
      default: ''
    },
    showCheckbox: {        // 是否显示checkbox
      type: Boolean,
      default: false
    },
    defaultExpandAll: {    // 默认展开全部节点
      type: Boolean,
      default: false
    },
    expandOnClickNode: {   // 点击节点时是否展开或折叠
      type: Boolean,
      default: true
    },
    checkOnClickNode: {    // 点击节点时是否选中节点
      type: Boolean,
      default: false
    },
    defaultExpandedKeys: { // 默认展开节点的keys
      type: Array,
      default: () => []
    },
    defaultCheckedKeys: {  // 默认选中节点的keys
      type: Array,
      default: () => []
    },
    props: {               // 配置项
      type: Object,
      default: () => {
        return {}
      }
    },
  },
  data () {
    return {
      deepData: '',
      isTree: true,
      defaultProps: {}
    }
  },
  computed: {
    _search () {
      return this.search.trim()
    },
    sourceData () {
      return deepCopy(this.data)
    }
  },
  watch: {
    sourceData () {
      this._initData()
    },
    _search (val) {
      if (val) return this.deepData = this._getLdqTree(this.deepData)
      const keys = this.getCheckedKeys()
      this.initData()
      this.setCheckedKeys(keys, true)
    }
  },
  created () {
    this.defaultProps = {
      name: this.props.name || 'name',
      children: this.props.children || 'children',
      disabled: this.props.disabled || 'disabled'
    }
    this._initData()
  },
  render () {
    return <div class="ldq-tree">
      { this.deepData.map(item => <search-node key={item[this.nodeKey]} data={item}></search-node>) }
    </div>
  },
  methods: {
    _initNode (node, parent) {
      const { name, children, disabled } = this.defaultProps
      const key = node[this.nodeKey]
      this.$set(node, name, node[name] || this.emptyText)
      this.$set(node, children, node[children] || [])
      this.$set(node, disabled, node[disabled] || false)
      this.$set(node, 'level', parent ? ~~parent.level + 1 : 1)
      this.$set(node, 'checked', Reflect.has(node, 'checked') ? node.checked : (parent && parent.checked) || this.defaultCheckedKeys.indexOf(key) > -1)
      this.$set(node, 'expand', Reflect.has(node, 'expand') ? node.expand : this.defaultExpandAll || this.defaultExpandedKeys.indexOf(key) > -1)
      this.$set(node, '$keys', [])
      this.$set(node, '$sort', 0)
    },
    _initData () { // 初始化数据
      const { children } = this.defaultProps
      const _deep = (arr, parent) => {
        arr.forEach(item => {
          this._initNode(item, parent)
          item[children]?.length && _deep(item[children], item)
          item.expand && parent && (parent.expand = true)
        })
      }
      _deep(this.sourceData)
      this.initData()
    },
    _getLdqTree (tree) { // 获取关键词索引并排序
      const { name, children } = this.defaultProps
      tree.forEach(item => {
        const keys = getDictionary(item[name], this._search)
        item.$keys = keys
        item.$sort = computSortNum(keys)
        if (item[children].length) item[children] = this._getLdqTree(item[children])
        item.$sort += item[children].reduce((max, item) => max > item.$sort ? max : item.$sort, 0)
      })
      return getSortData(tree)
    },
    _preorder (arr, callback) { // 前序迭代遍历
      if (!arr.length) return null
      const { children } = this.defaultProps
      let stack = [...arr]
      while (stack.length) {
        const curr = stack.shift()
        if (callback(curr)) return curr
        if (curr[children]?.length) stack.unshift(...curr[children])
      }
      return null
    },
    _downwardUpdateChecked (data, checked) { // 向下处理树节点的checked
      const { children, disabled } = this.defaultProps
      // 如果是末尾节点, 只需要判断disabled
      if (!data[children].length) return !data[disabled] && (data.checked = checked)
      // 过滤所有末尾节点, 如果全部为disabled就return
      if (!this._preorder(data[children], item => !item[children].length && !item[disabled])) return false
      // 否则还要判断children是否都是disable
      !data[disabled] && (data.checked = checked)
      data[children].forEach(item => this._downwardUpdateChecked(item, checked))
    },
    initData () { // 向外暴露一个初始化数据的方法
      const { children } = this.defaultProps
      const data = deepCopy(this.sourceData)
      this._preorder(data, item => !(item.$children = item[children]))
      this.deepData = this._getLdqTree(data)
    },
    getNode (key) { // 根据key获取对应深拷贝节点
      return deepCopy(this._getNode(key))
    },
    _getNode (key) { // 根据key获取对应引用节点
      return this._preorder(this.deepData, item => item[this.nodeKey] == key)
    },
    resetChecked () { // 取消所有节点的选中状态
      const { disabled } = this.defaultProps
      return !this._preorder(this.deepData, item => item.checked = false)
    },
    setCheckedKeys (keys, checked) { // 设置指定keys节点的checked
      keys = deepCopy(keys)
      this._preorder(this.deepData, item => {
        let index = keys.indexOf(item[this.nodeKey])
        if (index === -1) return false
        this._downwardUpdateChecked(item, checked)
        keys.splice(index, 1)
        return !keys.length
      })
    },
    getCheckedKeys () { // 获取所有选中节点的keys
      const keys = []
      this._preorder(this.deepData, item => item.checked && !keys.push(item[this.nodeKey]))
      return keys
    },
    getCheckedNodes () { // 获取所有选中节点的nodes
      const nodes = []
      this._preorder(this.deepData, item => item.checked && !nodes.push(item))
      return deepCopy(nodes)
    },
    remove (key) { // 删除指定的节点
      const curr = this._getNode(key)
      if (!curr) return !!console.warn('该节点不存在')
      const arr = curr.$pid ? this._getNode(curr.$pid)[this.defaultProps.children] : this.deepData
      arr.splice(arr.findIndex(item => item === curr), 1)
      return true
    },
    append (key, node) { // 添加新的子节点
      const nodeKey = node[this.nodeKey]
      if (!nodeKey) return !!console.warn('新节点没有node-key')
      if (key === null) {
        this._initNode(node, null)
        this.deepData.push(node)
        return true
      }
      if (this._getNode(nodeKey)) return !!console.warn('请勿重复添加新节点, node-key已存在')
      const curr = this._getNode(key)
      if (!curr) return !!console.warn('该节点不存在')
      this._initNode(node, curr)
      curr[this.defaultProps.children].push(node)
      return true
    },
    insertBefore (key, node) { // 向前添加兄弟节点
      const nodeKey = node[this.nodeKey]
      if (!nodeKey) return !!console.warn('新节点没有node-key')
      if (this._getNode(nodeKey)) return !!console.warn('请勿重复添加新节点, node-key已存在')
      const curr = this._getNode(key)
      if (!curr) return !!console.warn('该节点不存在')
      this._initNode(node, curr.$pid)
      const arr = curr.$pid ? this._getNode(curr.$pid)[this.defaultProps.children] : this.deepData
      arr.splice(arr.findIndex(item => item === curr), 0, node)
      return true
    },
    insertAfter (key, node) { // 向后添加兄弟节点
      const nodeKey = node[this.nodeKey]
      if (!nodeKey) return !!console.warn('新节点没有node-key')
      if (this._getNode(nodeKey)) return !!console.warn('请勿重复添加新节点, node-key已存在')
      const curr = this._getNode(key)
      if (!curr) return !!console.warn('该节点不存在')
      this._initNode(node, curr.$pid)
      const arr = curr.$pid ? this._getNode(curr.$pid)[this.defaultProps.children] : this.deepData
      arr.splice(arr.findIndex(item => item === curr) + 1, 0, node)
      return true
    }
  }
}
</script>

<style scoped>
  .ldq-tree {
    user-select: none;
  }
</style>
