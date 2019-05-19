<template>
  <div class="popup">
    <div class="popup-name">{{ data.name }}</div>
    <div class="popup-price">Price: {{ data.price }}</div>
    <!-- <div class="popup-location">Location: {{ data.location }}</div>
    <div class="popup-stall">{{ data.stall }}</div>-->
  </div>
</template>

<script>
export default {
  props: {
    data: {
      type: Object,
      default: () => ({
        name: "",
        stall: "",
        location: "",
        price: NaN
      })
    }
  }
};
</script>

<style lang="stylus">
$orange = #f2784b

.popup
  position absolute
  background #ffffff
  padding 15px
  opacity 0
  pointer-events none
  transition top .2s, left .2s, opacity .2s

  &.active
    opacity 1
    pointer-events auto

  // popup-location
  // popup-name
  &-location,
  &-name
    color #444

  // popup-stall
  &-stall
    color rgba(0, 0, 0, .7)

  // popup-price
  &-price
    color $orange

  &-location,
  &-name,
  &-stall,
  &-price
    .popup:not(.active) &
      color #fff
</style>