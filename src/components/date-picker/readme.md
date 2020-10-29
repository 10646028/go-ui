## `Vue`实战：日期选择器
> * [源码地址](https://github.com/wangkaiwd/go-ui/blob/master/src/components/date-picker/date-picker.vue)
> * [`demo`演示](http://localhost:8080/#/date-picker)

在日常工作中需要填写日期的时候，会用到日期选择器，来方便的进行日、月、年的选择。这里我们会用`Vue`来实现一个日期选择器，效果如下：
<p align="center">
  <img src="https://raw.githubusercontent.com/wangkaiwd/drawing-bed/master/Oct-29-2020%2014-48-17.gif" alt="">
</p>

实现功能： 
- [x] 日期选择弹出层
- [x] 选择天面板
- [x] 选择月面版
- [x] 选择年面版
- [x] 支持用户输入
- [x] `CSS`样式美化

组件的使用方式很简单，只需要传入对应的日期对象`value`即可：  
```vue
<template>
  <div class="date-picker">
    <go-date-picker v-model="value"></go-date-picker>
  </div>
</template>

<script>
export default {
  name: 'DatePicker',
  data () {
    return {
      value: undefined
    };
  },
};
</script>
```
下面就开始一步步实现组件吧😁！

### 日期选择弹出层
当用户点击输入框时，会弹出日期选择面板。在组件内部，会通过`visible`来控制弹出层的显示隐藏：  
```vue
<template>
  <div class="go-date-picker" ref="picker">
    <go-input
      class="go-date-picker-input"
      @focus="visible=true"
      prefix="calendar"
      placeholder="请选择时间"
    >
    </go-input>
    <div ref="popover" class="go-date-picker-popover" v-if="visible">
        <!-- day/month/year  panel    -->
    </div>
  </div>
</template>

<script>
export default {
  name: 'GoDatePicker',
  props: {
    value: {
      type: Date,
      default: () => new Date()
    }
  },
  data () {
    return {
      visible: false,
    };
  },
  mounted () {
    document.body.addEventListener('click', this.onClickBody);
  },
  beforeDestroy () {
    document.body.removeEventListener('click', this.onClickBody);
  },
  methods: {
    onClickBody (e) { // Vue内部会自动帮我们修改this指向
      const { picker, popover } = this.$refs;
      if (!popover) {return;}
      // 过滤掉弹出层和日期选择器内的元素
      if (picker.contains(e.target) || popover.contains(e.target)) {
        return;
      }
      this.visible = false;
    },
  }
};
</script>
```
当输入框激活时，显示弹出层，当点击外部区域时，会隐藏弹出层。需要注意的是当*点击`date-picker`内部，弹出层并不会隐藏*。

`Node.contains(otherNode)`可以用来判断`otherNode`是否是`Node`的后代节点(包括`Node`本身)，返回`Boolean`。这里我们通过这个`api`来判断点击的元素`e.target`是否在`date-picker`内部，如果是的话不会隐藏弹出层，可以让用户在`date-picker`进行相应的操作。

### 展示天面板
当用户点击输入框后，首先弹出的是天面板，面板头部会显示当前的年月信息。面板主体有6行，会分别包括上月、当前月、下月的天数： 
![](https://raw.githubusercontent.com/wangkaiwd/drawing-bed/master/20201029152243.png)

#### 显示头部信息
头部信息我们对传入的`value`进行拷贝，在内部通过`tempValue`来进行保存，并且监听`watch`的变化，保证`tempValue`可以获取到`value`的最新值。当我们在内部切换日期面板而并没有选中某个日期时，就不会更新`value`，而只是更新内部的`tempValue`属性：  
```vue
<script>
export default {
  name: 'GoDatePicker',
  props: {
    value: {
      type: Date,
      default: () => new Date()
    }
  },
  components: { PickerDays, PickerMonths, PickerYears },
  data () {
    return {
      visible: false,
      mode: 'picker-days',
      tempValue: cloneDate(this.value),
    };
  },
  computed: {
    formatDate () {
      const [year, month, day] = getYearMonthDay(this.tempValue);
      return { year, month: month + 1, day };
    },
  },
  watch: {
    value (val) {
      this.tempValue = cloneDate(val);
    }
  },
  // some code ...
};
</script>
```
`formatDate`计算属性会通过`tempValue`计算出当前的年、月、日，方便展示。

#### 显示内容区域
内容区域的展示会复杂很多，实现的思路如下：
* 获取当前月第一天是星期几，推导出前一个月展示的天数
* 获取当月的展示总天数
* 总共要展示的天数为42，减去前一个月和当前月展示的天数即为下个月展示的天数
```vue
<script>
export default {
  name: 'PickerDays',
  data () {
    return {
      weeks: ['一', '二', '三', '四', '五', '六', '日']
    };
  },
  // some code ...
  computed: {
    getDays () {
      const [year, month] = getYearMonthDay(this.tempValue);
      // 0 ~ 6, 需要往前推 startWeek + 1天
      const startWeek = new Date(year, month, 1).getDay();
      const prevLastDay = getPrevMonthLastDay(year, month);
      const curLastDay = getCurrentMonthLastDay(year, month);
      const days = [...this.getPrevMonthDays(prevLastDay, startWeek), ...this.getCurrentMonthDays(curLastDay), ...this.getNextMonthDays(curLastDay, startWeek)];
      // 转换成二维数组
      return toMatrix(days, 7);
    },
  },
  methods: {
    // 获取前一个月天数
    getPrevMonthDays (prevLastDay, startWeek) {
      const [year, month] = getYearMonthDay(this.tempValue);
      const prevMonthDays = [];
      for (let i = prevLastDay - startWeek + 1; i <= prevLastDay; i++) {
        prevMonthDays.push({
          date: new Date(year, month - 1, i),
          status: 'prev'
        });
      }
      return prevMonthDays;
    },
    // 获取当前月天数
    getCurrentMonthDays (curLastDay) {
      const [year, month] = getYearMonthDay(this.tempValue);
      const curMonthDays = [];
      for (let i = 1; i <= curLastDay; i++) {
        curMonthDays.push({
          date: new Date(year, month, i),
          status: 'current'
        });
      }
      return curMonthDays;
    },
    // 获取下一个月天数
    getNextMonthDays (curLastDay, startWeek) {
      const [year, month] = getYearMonthDay(this.tempValue);
      const nextMonthDays = [];
      for (let i = 1; i <= 42 - startWeek - curLastDay; i++) {
        nextMonthDays.push({
          date: new Date(year, month + 1, i),
          status: 'next'
        });
      }
      return nextMonthDays;
    },
    getDay (cell) {
      return cell.date.getDate();
    },
  }
};
</script>
```
我们将前一个月、当前月、下一个月的日期信息组成一个数组，然后转换位为拥有6个子数组，每个子数组中有7条信息的二维数组，方便遍历展示： 
```vue
<template>
  <div class="go-picker-days">
    <!--  some code ...  -->
    <div class="go-date-picker-days-row" v-for="(row,i) in getDays" :key="`${row}-${i}`">
      <div
        class="go-date-picker-days-cell"
        v-for="(cell,j) in row"
        :key="`${cell}-${j}`"
      >
        {{ getDay(cell) }}
      </div>
    </div>
  </div>
</template>
```
数组的格式如下： 
![](https://raw.githubusercontent.com/wangkaiwd/drawing-bed/master/20201029155114.png)

在计算日期时，如果传入的天数为0，则表示前一个月的最后一天。利用这个特性，可以节省我们很多的计算逻辑：  
```javascript
export const getCurrentMonthLastDay = (year, month) => {
  return new Date(year, month + 1, 0).getDate();
};

export const getPrevMonthLastDay = (year, month) => {
  return new Date(year, month, 0).getDate();
};
```
> * [Calculate last day of month in JavaScript](https://stackoverflow.com/a/13773408/12819402)
> * [How to get the last day of the previous month in Javascript or JQuery](https://stackoverflow.com/a/37803823/12819402)

在遍历展示的天的过程中，还可以通过日期信息来为其设置样式：  
```vue
<template>
  <div class="go-picker-days">
    <div class="go-date-picker-days-row" v-for="(row,i) in getDays" :key="`${row}-${i}`">
      <div
        class="go-date-picker-days-cell"
        :class="dayClasses(cell)"
        v-for="(cell,j) in row"
        :key="`${cell}-${j}`"
      >
        {{ getDay(cell) }}
      </div>
    </div>
  </div>
</template>

<script>
export default {
  // some code ...
  methods: {
    dayClasses (cell) {
      return {
        prev: cell.status === 'prev',
        next: cell.status === 'next',
        active: this.isSameDay(cell.date, this.value),
        today: this.isToday(cell.date)
      };
    },
    // 是否是选中的天
    isSameDay (date1, date2) {
      const [y1, m1, d1] = getYearMonthDay(date1);
      const [y2, m2, d2] = getYearMonthDay(date2);
      return y1 === y2 && m1 === m2 && d1 === d2;
    },
    // 是否是今天
    isToday (date) {
      const [y1, m1, d1] = getYearMonthDay(date);
      const [y2, m2, d2] = getYearMonthDay();
      return y1 === y2 && m1 === m2 && d1 === d2;
    }
  }
};
</script>
}
```
通过`dayClasses`方法，我们分别为添加如下`class`: 
* `prev`: 前一个月
* `next`: 下一个月
* `active`: 选中的日期
* `today`: 今天

之后便可以为这些不同状态分别添加不同的样式了。

#### 月份切换

### 展示月面板

### 展示年面板

### 
