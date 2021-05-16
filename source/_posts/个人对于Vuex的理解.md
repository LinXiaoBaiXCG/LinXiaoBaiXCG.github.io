---
title: 个人对于Vuex的理解
date: 2019-03-23 20:46:00
tags: [前端]
categories: 编程
---

## Vuex简介

Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。Vuex 也集成到 Vue 的官方调试工具 [devtools extension](https://github.com/vuejs/vue-devtools)，提供了诸如零配置的 time-travel 调试、状态快照导入导出等高级调试功能。

官方文档：[https://vuex.vuejs.org/zh/](https://vuex.vuejs.org/zh/)

## 个人理解

state:Vuex中的数据源，我们需要保存的数据就保存在这里。相当于vue中的data,用$store.state.modules名.xxx来获取

mutations:相当于vue中的methods,用this.$store.commit('事件名')来获取

actions:vuex中的actions是支持异步的,可理解为当组件需要调用多次mutations中的事件时,可将事件写入actions中,用$store.dispatch('事件名')来获取

getters:相当于vue中的computed计算属性。用于过滤,组合state。比如说state里面存了一个数组，数组有好多个数据，而我只想要用status：0的那些个，就可以用getters

modules:将store分割成模块，避免混淆

Vuex:可看作是个在vue中创建全局变量的东西,Vuex提供了一些优雅的方法，可以让我们改变全局变量的值

注:刷新页面Vuex会进行丢失数据，可配合session防止页面刷新丢失数据