<!--Todo:-->
<!--Export EmojiPicture-->
<!--Emoji Camera-->

<template>
    <div id="app">
        <md-content>
            <md-field>
                <label>Choose image</label>
                <md-file v-model="imageFile" accept="image/*" class="file-input" @change="loadImage()"/>
            </md-field>

            <md-list v-if="showOptions" class="md-double-line md-dense">
                <md-subheader>Options</md-subheader>

                <md-list-item>
                    <md-field>
                        <label>Emoji resolution</label>
                        <md-input v-model="emojiWidth" type="number"></md-input>
                    </md-field>
                </md-list-item>

                <md-list-item>
                    <md-switch v-model="useWebgl">Use WebGL renderer (faster, but no zoom capabilities)</md-switch>
                </md-list-item>

                <md-list-item>
                    <md-field>
                        <label for="palette-select">Quick palette</label>
                        <md-select id="palette-select" v-model="emojiPalette">
                            <md-option v-for="palette in paletteOptions" :value="palette.emojis">{{palette.name}}
                            </md-option>
                        </md-select>
                    </md-field>
                </md-list-item>
                <md-list-item>
                    <md-field>
                        <label>Palette</label>
                        <md-textarea v-model="emojiPalette"></md-textarea>
                    </md-field>
                </md-list-item>
                <!--<md-list-item>-->
                <!--<md-button class="md-dense md-raised" @click="updatePalette()">Update palette</md-button>-->
                <!--</md-list-item>-->
            </md-list>

            <md-button @click="showOptions=!showOptions">{{this.showOptions?"Hide options":"Show options"}}</md-button>

            <br>

            <md-button v-if="grid!==null" class="md-dense md-raised md-primary" @click="loadImage()">Update emoji
                picture
            </md-button>
            <md-progress-spinner v-show="loading" md-mode="indeterminate"></md-progress-spinner>
            <pre v-show="!useWebgl" class="output"></pre>
            <div v-show="useWebgl" class="render-output"></div>
            <img class="preview-image">
        </md-content>
    </div>
</template>

<script>
    import EmojiMapManager from '@/js/EmojiMapManager';
    import LabConvert from '@/js/LabConvert';
    import * as PIXI from 'pixi.js';
    import EmojiSplitter from "./js/EmojiSplitter";
    // import EmojiSplitter from "@/js/EmojiSplitter";
    // import EmojiMap from "@/js/EmojiMap";

    // let a = new EmojiMap('🏴🏁🚩🏳️‍🌈🇦🇫🇦🇽🇦🇱🇩🇿👌👌🏻');
    // console.log(a);

    export default {
        name: 'app',
        components: {},
        data() {
            return {
                useWebgl: false,
                showOptions: false,
                loading: false,
                imageFile: null,
                emojiMap: null,
                grid: null,
                imageData: null,
                emojiWidth: 100,
                spriteCache: {},
                timeouts: [],
                paletteOptions: [
                    {
                        name: 'Default',
                        emojis: `😂💯🙈😡😍😱👌👌🏻👌🏼👌🏽👌🏾👌🏿🤡😈🤢👹👾🤖👻💩💍💋👅👄👥👀🎒👛👘👒👗👔👖👚👠🐾🌴🍃🌏🌓🌝🍏🍎🍋🥝🍳🥚🍖🏀⚾🎾🏐🏉👣💛💚💙💜🖤`
                    },
                    {
                        name: 'Flags',
                        emojis: '🏳️🏴🏁🚩🏳️‍🌈🇦🇫🇦🇽🇦🇱🇩🇿🇦🇸🇦🇩🇦🇴🇦🇮🇦🇶🇦🇬🇦🇷🇦🇲🇦🇼🇦🇺🇦🇹🇦🇿🇧🇸🇧🇭🇧🇩🇧🇧🇧🇾🇧🇪🇧🇿🇧🇯🇧🇲🇧🇹🇧🇴🇧🇦🇧🇼🇧🇷🇮🇴🇻🇬🇧🇳🇧🇬🇧🇫🇧🇮🇰🇭🇨🇲🇨🇦🇮🇨🇨🇻🇧🇶🇰🇾🇨🇫🇹🇩🇨🇱🇨🇳🇨🇽🇨🇨🇨🇴🇰🇲🇨🇬🇨🇩🇨🇰🇨🇷🇨🇮🇭🇷🇨🇺🇨🇼🇨🇾🇨🇿🇩🇰🇩🇯🇩🇲🇩🇴🇪🇨🇪🇬🇸🇻🇬🇶🇪🇷🇪🇪🇪🇹🇪🇺🇫🇰🇫🇴🇫🇯🇫🇮🇫🇷🇬🇫🇵🇫🇹🇫🇬🇦🇬🇲🇬🇪🇩🇪🇬🇭🇬🇮🇬🇷🇬🇱🇬🇩🇬🇵🇬🇺🇬🇹🇬🇬🇬🇳🇬🇼🇬🇾🇭🇹🇭🇳🇭🇰🇭🇺🇮🇸🇮🇳🇮🇩🇮🇷🇮🇶🇮🇪🇮🇲🇮🇱🇮🇹🇯🇲🇯🇵🎌🇯🇪🇯🇴🇰🇿🇰🇪🇰🇮🇽🇰🇰🇼🇰🇬🇱🇦🇱🇻🇱🇧🇱🇸🇱🇷🇱🇾🇱🇮🇱🇹🇱🇺🇲🇴🇲🇰🇲🇬🇲🇼🇲🇾🇲🇻🇲🇱🇲🇹🇲🇭🇲🇶🇲🇷🇲🇺🇾🇹🇲🇽🇫🇲🇲🇩🇲🇨🇲🇳🇲🇪🇲🇸🇲🇦🇲🇿🇲🇲🇳🇦🇳🇷🇳🇵🇳🇱🇳🇨🇳🇿🇳🇮🇳🇪🇳🇬🇳🇺🇳🇫🇰🇵🇲🇵🇳🇴🇴🇲🇵🇰🇵🇼🇵🇸🇵🇦🇵🇬🇵🇾🇵🇪🇵🇭🇵🇳🇵🇱🇵🇹🇵🇷🇶🇦🇷🇪🇷🇴🇷🇺🇷🇼🇼🇸🇸🇲🇸🇦🇸🇳🇷🇸🇸🇨🇸🇱🇸🇬🇸🇽🇸🇰🇸🇮🇬🇸🇸🇧🇸🇴🇿🇦🇰🇷🇸🇸🇪🇸🇱🇰🇧🇱🇸🇭🇰🇳🇱🇨🇵🇲🇻🇨🇸🇩🇸🇷🇸🇿🇸🇪🇨🇭🇸🇾🇹🇼🇹🇯🇹🇿🇹🇭🇹🇱🇹🇬🇹🇰🇹🇴🇹🇹🇹🇳🇹🇷🇹🇲🇹🇨🇹🇻🇻🇮🇺🇬🇺🇦🇦🇪🇬🇧🇺🇸🇺🇾🇺🇿🇻🇺🇻🇦🇻🇪🇻🇳🇼🇫🇪🇭🇾🇪🇿🇲🇿🇼'
                    },
                    {name: 'Sepia', emojis: `👌👌🏻👌🏼👌🏽👌🏾👌`},
                    {
                        name: 'Travel',
                        emojis: `🚗🚕🚙🚌🚎🏎🚓🚑🚒🚐🚚🚛🚜🛴🚲🛵🏍🚨🚔🚍🚘🚖🚡🚠🚟🚃🚋🚞🚝🚄🚅🚈🚂🚆🚇🚊🚉🚁🛩✈️🛫🛬🚀🛰💺🛶⛵️🛥🚤🛳⛴🚢⚓️🚧⛽️🚏🚦🚥🗺🗿🗽⛲️🗼🏰🏯🏟🎡🎢🎠⛱🏖🏝⛰🏔🗻🌋🏜🏕⛺️🛤🛣🏗🏭🏠🏡🏘🏚🏢🏬🏣🏤🏥🏦🏨🏪🏫🏩💒🏛⛪️🕌🕍🕋⛩🗾🎑🏞🌅🌄🌠🎇🎆🌇🌆🏙🌃🌌🌉🌁`
                    },
                    {
                        name: 'All (very slow)',
                        emojis: `😀😁😂🤣😃😄😅😆😉😊😋😎😍😘😗😙😚☺️🙂🤗🤩🤔🤨😐😑😶🙄😏😣😥😮🤐😯😪😫😴😌😛😜😝🤤😒😓😔😕🙃🤑😲☹️🙁😖😞😟😤😢😭😦😧😨😩🤯😬😰😱😳🤪😵😡😠🤬😷🤒🤕🤢🤮🤧😇🤠🤡🤥🤫🤭🧐🤓😈👿👹👺💀👻👽🤖💩😺😸😹😻😼😽🙀😿😾👶👦👧👨👩👴👵👨‍⚕️👩‍⚕️👨‍🎓👩‍🎓👨‍⚖️👩‍⚖️👨‍🌾👩‍🌾👨‍🍳👩‍🍳👨‍🔧👩‍🔧👨‍🏭👩‍🏭👨‍💼👩‍💼👨‍🔬👩‍🔬👨‍💻👩‍💻👨‍🎤👩‍🎤👨‍🎨👩‍🎨👨‍✈️👩‍✈️👨‍🚀👩‍🚀👨‍🚒👩‍🚒👮👮‍♂️👮‍♀️🕵🕵️‍♂️🕵️‍♀️💂💂‍♂️💂‍♀️👷👷‍♂️👷‍♀️🤴👸👳👳‍♂️👳‍♀️👲🧕🧔👱👱‍♂️👱‍♀️🤵👰🤰🤱👼🎅🤶🧙‍♀️🧙‍♂️🧚‍♀️🧚‍♂️🧛‍♀️🧛‍♂️🧜‍♀️🧜‍♂️🧝‍♀️🧝‍♂️🧞‍♀️🧞‍♂️🧟‍♀️🧟‍♂️🙍🙍‍♂️🙍‍♀️🙎🙎‍♂️🙎‍♀️🙅🙅‍♂️🙅‍♀️🙆🙆‍♂️🙆‍♀️💁💁‍♂️💁‍♀️🙋🙋‍♂️🙋‍♀️🙇🙇‍♂️🙇‍♀️🤦🤦‍♂️🤦‍♀️🤷🤷‍♂️🤷‍♀️💆💆‍♂️💆‍♀️💇💇‍♂️💇‍♀️🚶🚶‍♂️🚶‍♀️🏃🏃‍♂️🏃‍♀️💃🕺👯👯‍♂️👯‍♀️🧖‍♀️🧖‍♂️🕴🗣👤👥👫👬👭💏👨‍❤️‍💋‍👨👩‍❤️‍💋‍👩💑👨‍❤️‍👨👩‍❤️‍👩👪👨‍👩‍👦👨‍👩‍👧👨‍👩‍👧‍👦👨‍👩‍👦‍👦👨‍👩‍👧‍👧👨‍👨‍👦👨‍👨‍👧👨‍👨‍👧‍👦👨‍👨‍👦‍👦👨‍👨‍👧‍👧👩‍👩‍👦👩‍👩‍👧👩‍👩‍👧‍👦👩‍👩‍👦‍👦👩‍👩‍👧‍👧👨‍👦👨‍👦‍👦👨‍👧👨‍👧‍👦👨‍👧‍👧👩‍👦👩‍👦‍👦👩‍👧👩‍👧‍👦👩‍👧‍👧🤳💪👈👉☝️👆🖕👇✌️🤞🖖🤘🖐✋👌👍👎✊👊🤛🤜🤚👋🤟✍️👏👐🙌🤲🙏🤝💅👂👃👣👀👁🧠👅👄💋👓🕶👔👕👖🧣🧤🧥🧦👗👘👙👚👛👜👝🎒👞👟👠👡👢👑👒🎩🎓🧢⛑💄💍🌂💼👐🏻🙌🏻👏🏻🙏🏻👍🏻👎🏻👊🏻✊🏻🤛🏻🤜🏻🤞🏻✌🏻🤘🏻👌🏻👈🏻👉🏻👆🏻👇🏻☝🏻✋🏻🤚🏻🖐🏻🖖🏻👋🏻🤙🏻💪🏻🖕🏻✍🏻🤳🏻💅🏻👂🏻👃🏻👶🏻👦🏻👧🏻👨🏻👩🏻👱🏻‍♀️👱🏻👴🏻👵🏻👲🏻👳🏻‍♀️👳🏻👮🏻‍♀️👮🏻👷🏻‍♀️👷🏻💂🏻‍♀️💂🏻🕵🏻‍♀️🕵🏻👩🏻‍⚕️👨🏻‍⚕️👩🏻‍🌾👨🏻‍🌾👩🏻‍🍳👨🏻‍🍳👩🏻‍🎓👨🏻‍🎓👩🏻‍🎤👨🏻‍🎤👩🏻‍🏫👨🏻‍🏫👩🏻‍🏭👨🏻‍🏭👩🏻‍💻👨🏻‍💻👩🏻‍💼👨🏻‍💼👩🏻‍🔧👨🏻‍🔧👩🏻‍🔬👨🏻‍🔬👩🏻‍🎨👨🏻‍🎨👩🏻‍🚒👨🏻‍🚒👩🏻‍✈️👨🏻‍✈️👩🏻‍🚀👨🏻‍🚀👩🏻‍⚖️👨🏻‍⚖️🤶🏻🎅🏻👸🏻🤴🏻👰🏻🤵🏻👼🏻🤰🏻🙇🏻‍♀️🙇🏻💁🏻💁🏻‍♂️🙅🏻🙅🏻‍♂️🙆🏻🙆🏻‍♂️🙋🏻🙋🏻‍♂️🤦🏻‍♀️🤦🏻‍♂️🤷🏻‍♀️🤷🏻‍♂️🙎🏻🙎🏻‍♂️🙍🏻🙍🏻‍♂️💇🏻💇🏻‍♂️💆🏻💆🏻‍♂️🕴🏻💃🏻🕺🏻🚶🏻‍♀️🚶🏻🏃🏻‍♀️🏃🏻🏋🏻‍♀️🏋🏻🤸🏻‍♀️🤸🏻‍♂️⛹🏻‍♀️⛹🏻🤾🏻‍♀️🤾🏻‍♂️🏌🏻‍♀️🏌🏻🏄🏻‍♀️🏄🏻🏊🏻‍♀️🏊🏻🤽🏻‍♀️🤽🏻‍♂️🚣🏻‍♀️🚣🏻🏇🏻🚴🏻‍♀️🚴🏻🚵🏻‍♀️🚵🏻🤹🏻‍♀️🤹🏻‍♂️🛀🏻👐🏼🙌🏼👏🏼🙏🏼👍🏼👎🏼👊🏼✊🏼🤛🏼🤜🏼🤞🏼✌🏼🤘🏼👌🏼👈🏼👉🏼👆🏼👇🏼☝🏼✋🏼🤚🏼🖐🏼🖖🏼👋🏼🤙🏼💪🏼🖕🏼✍🏼🤳🏼💅🏼👂🏼👃🏼👶🏼👦🏼👧🏼👨🏼👩🏼👱🏼‍♀️👱🏼👴🏼👵🏼👲🏼👳🏼‍♀️👳🏼👮🏼‍♀️👮🏼👷🏼‍♀️👷🏼💂🏼‍♀️💂🏼🕵🏼‍♀️🕵🏼👩🏼‍⚕️👨🏼‍⚕️👩🏼‍🌾👨🏼‍🌾👩🏼‍🍳👨🏼‍🍳👩🏼‍🎓👨🏼‍🎓👩🏼‍🎤👨🏼‍🎤👩🏼‍🏫👨🏼‍🏫👩🏼‍🏭👨🏼‍🏭👩🏼‍💻👨🏼‍💻👩🏼‍💼👨🏼‍💼👩🏼‍🔧👨🏼‍🔧👩🏼‍🔬👨🏼‍🔬👩🏼‍🎨👨🏼‍🎨👩🏼‍🚒👨🏼‍🚒👩🏼‍✈️👨🏼‍✈️👩🏼‍🚀👨🏼‍🚀👩🏼‍⚖️👨🏼‍⚖️🤶🏼🎅🏼👸🏼🤴🏼👰🏼🤵🏼👼🏼🤰🏼🙇🏼‍♀️🙇🏼💁🏼💁🏼‍♂️🙅🏼🙅🏼‍♂️🙆🏼🙆🏼‍♂️🙋🏼🙋🏼‍♂️🤦🏼‍♀️🤦🏼‍♂️🤷🏼‍♀️🤷🏼‍♂️🙎🏼🙎🏼‍♂️🙍🏼🙍🏼‍♂️💇🏼💇🏼‍♂️💆🏼💆🏼‍♂️🕴🏼💃🏼🕺🏼🚶🏼‍♀️🚶🏼🏃🏼‍♀️🏃🏼🏋🏼‍♀️🏋🏼🤸🏼‍♀️🤸🏼‍♂️⛹🏼‍♀️⛹🏼🤾🏼‍♀️🤾🏼‍♂️🏌🏼‍♀️🏌🏼🏄🏼‍♀️🏄🏼🏊🏼‍♀️🏊🏼🤽🏼‍♀️🤽🏼‍♂️🚣🏼‍♀️🚣🏼🏇🏼🚴🏼‍♀️🚴🏼🚵🏼‍♀️🚵🏻🤹🏼‍♀️🤹🏼‍♂️🛀🏼👐🏽🙌🏽👏🏽🙏🏽👍🏽👎🏽👊🏽✊🏽🤛🏽🤜🏽🤞🏽✌🏽🤘🏽👌🏽👈🏽👉🏽👆🏽👇🏽☝🏽✋🏽🤚🏽🖐🏽🖖🏽👋🏽🤙🏽💪🏽🖕🏽✍🏽🤳🏽💅🏽👂🏽👃🏽👶🏽👦🏽👧🏽👨🏽👩🏽👱🏽‍♀️👱🏽👴🏽👵🏽👲🏽👳🏽‍♀️👳🏽👮🏽‍♀️👮🏽👷🏽‍♀️👷🏽💂🏽‍♀️💂🏽🕵🏽‍♀️🕵🏽👩🏽‍⚕️👨🏽‍⚕️👩🏽‍🌾👨🏽‍🌾👩🏽‍🍳👨🏽‍🍳👩🏽‍🎓👨🏽‍🎓👩🏽‍🎤👨🏽‍🎤👩🏽‍🏫👨🏽‍🏫👩🏽‍🏭👨🏽‍🏭👩🏽‍💻👨🏽‍💻👩🏽‍💼👨🏽‍💼👩🏽‍🔧👨🏽‍🔧👩🏽‍🔬👨🏽‍🔬👩🏽‍🎨👨🏽‍🎨👩🏽‍🚒👨🏽‍🚒👩🏽‍✈️👨🏽‍✈️👩🏽‍🚀👨🏽‍🚀👩🏽‍⚖️👨🏽‍⚖️🤶🏽🎅🏽👸🏽🤴🏽👰🏽🤵🏽👼🏽🤰🏽🙇🏽‍♀️🙇🏽💁🏽💁🏽‍♂️🙅🏽🙅🏽‍♂️🙆🏽🙆🏽‍♂️🙋🏽🙋🏽‍♂️🤦🏽‍♀️🤦🏽‍♂️🤷🏽‍♀️🤷🏽‍♂️🙎🏽🙎🏽‍♂️🙍🏽🙍🏽‍♂️💇🏽💇🏽‍♂️💆🏽💆🏽‍♂️🕴🏼💃🏽🕺🏽🚶🏽‍♀️🚶🏽🏃🏽‍♀️🏃🏽🏋🏽‍♀️🏋🏽🤸🏽‍♀️🤸🏽‍♂️⛹🏽‍♀️⛹🏽🤾🏽‍♀️🤾🏽‍♂️🏌🏽‍♀️🏌🏽🏄🏽‍♀️🏄🏽🏊🏽‍♀️🏊🏽🤽🏽‍♀️🤽🏽‍♂️🚣🏽‍♀️🚣🏽🏇🏽🚴🏽‍♀️🚴🏽🚵🏽‍♀️🚵🏽🤹🏽‍♀️🤹🏽‍♂️🛀🏽👐🏾🙌🏾👏🏾🙏🏾👍🏾👎🏾👊🏾✊🏾🤛🏾🤜🏾🤞🏾✌🏾🤘🏾👌🏾👈🏾👉🏾👆🏾👇🏾☝🏾✋🏾🤚🏾🖐🏾🖖🏾👋🏾🤙🏾💪🏾🖕🏾✍🏾🤳🏾💅🏾👂🏾👃🏾👶🏾👦🏾👧🏾👨🏾👩🏾👱🏾‍♀️👱🏾👴🏾👵🏾👲🏾👳🏾‍♀️👳🏾👮🏾‍♀️👮🏾👷🏾‍♀️👷🏾💂🏾‍♀️💂🏾🕵🏾‍♀️🕵🏾👩🏾‍⚕️👨🏾‍⚕️👩🏾‍🌾👨🏾‍🌾👩🏾‍🍳👨🏾‍🍳👩🏾‍🎓👨🏾‍🎓👩🏾‍🎤👨🏾‍🎤👩🏾‍🏫👨🏾‍🏫👩🏾‍🏭👨🏾‍🏭👩🏾‍💻👨🏾‍💻👩🏾‍💼👨🏾‍💼👩🏾‍🔧👨🏾‍🔧👩🏾‍🔬👨🏾‍🔬👩🏾‍🎨👨🏾‍🎨👩🏾‍🚒👨🏾‍🚒👩🏾‍✈️👨🏾‍✈️👩🏾‍🚀👨🏾‍🚀👩🏾‍⚖️👨🏾‍⚖️🤶🏾🎅🏾👸🏾🤴🏾👰🏾🤵🏾👼🏾🤰🏾🙇🏾‍♀️🙇🏾💁🏾💁🏾‍♂️🙅🏾🙅🏾‍♂️🙆🏾🙆🏾‍♂️🙋🏾🙋🏾‍♂️🤦🏾‍♀️🤦🏾‍♂️🤷🏾‍♀️🤷🏾‍♂️🙎🏾🙎🏾‍♂️🙍🏾🙍🏾‍♂️💇🏾💇🏾‍♂️💆🏾💆🏾‍♂️🕴🏾💃🏾🕺🏾🚶🏾‍♀️🚶🏾🏃🏾‍♀️🏃🏾🏋🏾‍♀️🏋🏾🤸🏾‍♀️🤸🏾‍♂️⛹🏾‍♀️⛹🏾🤾🏾‍♀️🤾🏾‍♂️🏌🏾‍♀️🏌🏾🏄🏾‍♀️🏄🏾🏊🏾‍♀️🏊🏾🤽🏾‍♀️🤽🏾‍♂️🚣🏾‍♀️🚣🏾🏇🏾🚴🏾‍♀️🚴🏾🚵🏾‍♀️🚵🏾🤹🏾‍♀️🤹🏾‍♂️🛀🏾👐🏿🙌🏿👏🏿🙏🏿👍🏿👎🏿👊🏿✊🏿🤛🏿🤜🏿🤞🏿✌🏿🤘🏿👌🏿👈🏿👉🏿👆🏿👇🏿☝🏿✋🏿🤚🏿🖐🏿🖖🏿👋🏿🤙🏿💪🏿🖕🏿✍🏿🤳🏿💅🏿👂🏿👃🏿👶🏿👦🏿👧🏿👨🏿👩🏿👱🏿‍♀️👱🏿👴🏿👵🏿👲🏿👳🏿‍♀️👳🏿👮🏿‍♀️👮🏿👷🏿‍♀️👷🏿💂🏿‍♀️💂🏿🕵🏿‍♀️🕵🏿👩🏿‍⚕️👨🏿‍⚕️👩🏿‍🌾👨🏿‍🌾👩🏿‍🍳👨🏿‍🍳👩🏿‍🎓👨🏿‍🎓👩🏿‍🎤👨🏿‍🎤👩🏿‍🏫👨🏿‍🏫👩🏿‍🏭👨🏿‍🏭👩🏿‍💻👨🏿‍💻👩🏿‍💼👨🏿‍💼👩🏿‍🔧👨🏿‍🔧👩🏿‍🔬👨🏿‍🔬👩🏿‍🎨👨🏿‍🎨👩🏿‍🚒👨🏿‍🚒👩🏿‍✈️👨🏿‍✈️👩🏿‍🚀👨🏿‍🚀👩🏿‍⚖️👨🏿‍⚖️🤶🏿🎅🏿👸🏿🤴🏿👰🏿🤵🏿👼🏿🤰🏿🙇🏿‍♀️🙇🏿💁🏿💁🏿‍♂️🙅🏿🙅🏿‍♂️🙆🏿🙆🏿‍♂️🙋🏿🙋🏿‍♂️🤦🏿‍♀️🤦🏿‍♂️🤷🏿‍♀️🤷🏿‍♂️🙎🏿🙎🏿‍♂️🙍🏿🙍🏿‍♂️💇🏿💇🏿‍♂️💆🏿💆🏿‍♂️🕴🏿💃🏿🕺🏿🚶🏿‍♀️🚶🏿🏃🏿‍♀️🏃🏿🏋🏿‍♀️🏋🏿🤸🏿‍♀️🤸🏿‍♂️⛹🏿‍♀️⛹🏿🤾🏿‍♀️🤾🏿‍♂️🏌🏿‍♀️🏌🏿🏄🏿‍♀️🏄🏿🏊🏿‍♀️🏊🏿🤽🏿‍♀️🤽🏿‍♂️🚣🏿‍♀️🚣🏿🏇🏿🚴🏿‍♀️🚴🏿🚵🏿‍♀️🚵🏿🤹🏿‍♀️🤹🏿‍♂️🛀🏿🐶🐱🐭🐹🐰🦊🐻🐼🐨🐯🦁🐮🐷🐽🐸🐵🙊🙉🙊🐒🐔🐧🐦🐤🐣🐥🦆🦅🦉🦇🐺🐗🐴🦄🐝🐛🦋🐌🐚🐞🐜🕷🕸🐢🐍🦎🦂🦀🦑🐙🦐🐠🐟🐡🐬🦈🐳🐋🐊🐆🐅🐃🐂🐄🦌🐪🐫🐘🦏🦍🐎🐖🐐🐏🐑🐕🐩🐈🐓🦃🕊🐇🐁🐀🐿🐾🐉🐲🌵🎄🌲🌳🌴🌱🌿☘️🍀🎍🎋🍃🍂🍁🍄🌾💐🌷🌹🥀🌻🌼🌸🌺🌎🌍🌏🌕🌖🌗🌘🌑🌒🌓🌔🌚🌝🌞🌛🌜🌙💫⭐️🌟✨⚡️🔥💥☄️☀️🌤⛅️🌥🌦🌈☁️🌧⛈🌩🌨☃️⛄️❄️🌬💨🌪🌫🌊💧💦☔️🍏🍎🍐🍊🍋🍌🍉🍇🍓🍈🍒🍑🍍🥝🥑🍅🍆🥒🥕🌽🌶🥔🍠🌰🥜🍯🥐🍞🥖🧀🥚🍳🥓🥞🍤🍗🍖🍕🌭🍔🍟🥙🌮🌯🥗🥘🍝🍜🍲🍥🍣🍱🍛🍚🍙🍘🍢🍡🍧🍨🍦🍰🎂🍮🍭🍬🍫🍿🍩🍪🥛🍼☕️🍵🍶🍺🍻🥂🍷🥃🍸🍹🍾🥄🍴🍽⚽️🏀🏈⚾️🎾🏐🏉🎱🏓🏸🥅🏒🏑🏏⛳️🏹🎣🥊🥋⛸🎿⛷🏂🏋️‍♀️🏋️🤺🤼‍♀️🤼‍♂️🤸‍♀️🤸‍♂️⛹️‍♀️⛹️🤾‍♀️🤾‍♂️🏌️‍♀️🏌️🏄‍♀️🏄🏊‍♀️🏊🤽‍♀️🤽‍♂️🚣‍♀️🚣🏇🚴‍♀️🚴🚵‍♀️🚵🎽🏅🎖🥇🥈🥉🏆🏵🎗🎫🎟🎪🤹‍♀️🤹‍♂️🎭🎨🎬🎤🎧🎼🎹🥁🎷🎺🎸🎻🎲🎯🎳🎮🎰🚗🚕🚙🚌🚎🏎🚓🚑🚒🚐🚚🚛🚜🛴🚲🛵🏍🚨🚔🚍🚘🚖🚡🚠🚟🚃🚋🚞🚝🚄🚅🚈🚂🚆🚇🚊🚉🚁🛩✈️🛫🛬🚀🛰💺🛶⛵️🛥🚤🛳⛴🚢⚓️🚧⛽️🚏🚦🚥🗺🗿🗽⛲️🗼🏰🏯🏟🎡🎢🎠⛱🏖🏝⛰🏔🗻🌋🏜🏕⛺️🛤🛣🏗🏭🏠🏡🏘🏚🏢🏬🏣🏤🏥🏦🏨🏪🏫🏩💒🏛⛪️🕌🕍🕋⛩🗾🎑🏞🌅🌄🌠🎇🎆🌇🌆🏙🌃🌌🌉🌁⌚️📱📲💻⌨️🖥🖨🖱🖲🕹🗜💽💾💿📀📼📷📸📹🎥📽🎞📞☎️📟📠📺📻🎙🎚🎛⏱⏲⏰🕰⌛️⏳📡🔋🔌💡🔦🕯🗑🛢💸💵💴💶💷💰💳💎⚖️🔧🔨⚒🛠⛏🔩⚙️⛓🔫💣🔪🗡⚔️🛡🚬⚰️⚱️🏺🔮📿💈⚗️🔭🔬🕳💊💉🌡🚽🚰🚿🛁🛀🛎🔑🗝🚪🛋🛏🛌🖼🛍🛒🎁🎈🎏🎀🎊🎉🎎🏮🎐✉️📩📨📧💌📥📤📦🏷📪📫📬📭📮📯📜📃📄📑📊📈📉🗒🗓📆📅📇🗃🗳🗄📋📁📂🗂🗞📰📓📔📒📕📗📘📙📚📖🔖🔗📎🖇📐📏📌📍📌🎌🏳️🏴🏁🏳️‍🌈✂️🖊🖋✒️🖌🖍📝✏️🔍🔎🔏🔐🔒🔓❤️💛💚💙💜🖤💔❣️💕💞💓💗💖💘💝💟☮️✝️☪️🕉☸️✡️🔯🕎☯️☦️🛐⛎♈️♉️♊️♋️♌️♍️♎️♏️♐️♑️♒️♓️🆔⚛️🉑☢️☣️📴📳🈶🈚️🈸🈺🈷️✴️🆚💮🉐㊙️㊗️🈴🈵🈹🈲🅰️🅱️🆎🆑🅾️🆘❌⭕️🛑⛔️📛🚫💯💢♨️🚷🚯🚳🚱🔞📵🚭❗️❕❓❔‼️⁉️🔅🔆〽️⚠️🚸🔱⚜️🔰♻️✅🈯️💹❇️✳️❎🌐💠Ⓜ️🌀💤🏧🚾♿️🅿️🈳🈂️🛂🛃🛄🛅🚹🚺🚼🚻🚮🎦📶🈁🔣ℹ️🔤🔡🔠🆖🆗🆙🆒🆕🆓0️⃣1️⃣2️⃣3️⃣4️⃣5️⃣6️⃣7️⃣8️⃣9️⃣🔟🔢#️⃣*️⃣▶️⏸⏯⏹⏺⏭⏮⏩⏪⏫⏬◀️🔼🔽➡️⬅️⬆️⬇️↗️↘️↙️↖️↕️↔️↪️↩️⤴️⤵️🔀🔁🔂🔄🔃🎵🎶➕➖➗✖️💲💱™️©️®️〰️➰➿🔚🔙🔛🔝✔️☑️🔘⚪️⚫️🔴🔵🔺🔻🔸🔹🔶🔷🔳🔲▪️▫️◾️◽️◼️◻️⬛️⬜️🔈🔇🔉🔊🔔🔕📣📢👁‍🗨💬💭🗯♠️♣️♥️♦️🃏🎴🀄️🕐🕑🕒🕓🕔🕕🕖🕗🕘🕙🕚🕛🕜🕝🕞🕟🕠🕡🕢🕣🕤🕥🕦🕧🏳️🏴🏁🚩🏳️‍🌈🇦🇫🇦🇽🇦🇱🇩🇿🇦🇸🇦🇩🇦🇴🇦🇮🇦🇶🇦🇬🇦🇷🇦🇲🇦🇼🇦🇺🇦🇹🇦🇿🇧🇸🇧🇭🇧🇩🇧🇧🇧🇾🇧🇪🇧🇿🇧🇯🇧🇲🇧🇹🇧🇴🇧🇦🇧🇼🇧🇷🇮🇴🇻🇬🇧🇳🇧🇬🇧🇫🇧🇮🇰🇭🇨🇲🇨🇦🇮🇨🇨🇻🇧🇶🇰🇾🇨🇫🇹🇩🇨🇱🇨🇳🇨🇽🇨🇨🇨🇴🇰🇲🇨🇬🇨🇩🇨🇰🇨🇷🇨🇮🇭🇷🇨🇺🇨🇼🇨🇾🇨🇿🇩🇰🇩🇯🇩🇲🇩🇴🇪🇨🇪🇬🇸🇻🇬🇶🇪🇷🇪🇪🇪🇹🇪🇺🇫🇰🇫🇴🇫🇯🇫🇮🇫🇷🇬🇫🇵🇫🇹🇫🇬🇦🇬🇲🇬🇪🇩🇪🇬🇭🇬🇮🇬🇷🇬🇱🇬🇩🇬🇵🇬🇺🇬🇹🇬🇬🇬🇳🇬🇼🇬🇾🇭🇹🇭🇳🇭🇰🇭🇺🇮🇸🇮🇳🇮🇩🇮🇷🇮🇶🇮🇪🇮🇲🇮🇱🇮🇹🇯🇲🇯🇵🎌🇯🇪🇯🇴🇰🇿🇰🇪🇰🇮🇽🇰🇰🇼🇰🇬🇱🇦🇱🇻🇱🇧🇱🇸🇱🇷🇱🇾🇱🇮🇱🇹🇱🇺🇲🇴🇲🇰🇲🇬🇲🇼🇲🇾🇲🇻🇲🇱🇲🇹🇲🇭🇲🇶🇲🇷🇲🇺🇾🇹🇲🇽🇫🇲🇲🇩🇲🇨🇲🇳🇲🇪🇲🇸🇲🇦🇲🇿🇲🇲🇳🇦🇳🇷🇳🇵🇳🇱🇳🇨🇳🇿🇳🇮🇳🇪🇳🇬🇳🇺🇳🇫🇰🇵🇲🇵🇳🇴🇴🇲🇵🇰🇵🇼🇵🇸🇵🇦🇵🇬🇵🇾🇵🇪🇵🇭🇵🇳🇵🇱🇵🇹🇵🇷🇶🇦🇷🇪🇷🇴🇷🇺🇷🇼🇼🇸🇸🇲🇸🇦🇸🇳🇷🇸🇸🇨🇸🇱🇸🇬🇸🇽🇸🇰🇸🇮🇬🇸🇸🇧🇸🇴🇿🇦🇰🇷🇸🇸🇪🇸🇱🇰🇧🇱🇸🇭🇰🇳🇱🇨🇵🇲🇻🇨🇸🇩🇸🇷🇸🇿🇸🇪🇨🇭🇸🇾🇹🇼🇹🇯🇹🇿🇹🇭🇹🇱🇹🇬🇹🇰🇹🇴🇹🇹🇹🇳🇹🇷🇹🇲🇹🇨🇹🇻🇻🇮🇺🇬🇺🇦🇦🇪🇬🇧🇺🇸🇺🇾🇺🇿🇻🇺🇻🇦🇻🇪🇻🇳🇼🇫🇪🇭🇾🇪🇿🇲🇿🇼😃💁People•🐻🌻Animals•🍔🍹Food•🎷⚽️Activities•🚘🌇Travel•💡🎉Objects•💖🔣Symbols•🎌🏳️‍🌈Flags🤩🤨🤯🤪🤬🤮🤫🤭🧐🧒🧑🧓🧕🧔🤱🧙🧙‍♀️🧙‍♂️🧚🧚‍♀️🧚‍♂️🧛🧛‍♀️🧛‍♂️🧜🧜‍♀️🧜‍♂️🧝🧝‍♀️🧝‍♂️🧞🧞‍♀️🧞‍♂️🧟🧟‍♀️🧟‍♂️🧖🧖‍♀️🧖‍♂️🧗🧗‍♀️🧗‍♂️🧘🧘‍♀️🧘‍♂️🤟🤲🧠🧡🧣🧤🧥🧦🧢🦓🦒🦔🦕🦖🦗🥥🥦🥨🥩🥪🥣🥫🥟🥠🥡🥧🥤🥢🛸🛷🥌🏴󠁧󠁢󠁥󠁮󠁧󠁿🏴󠁧󠁢󠁳󠁣󠁴󠁿🏴󠁧󠁢󠁷󠁬󠁳󠁿😃💁People•🐻🌻Animals•🍔🍹Food•🎷⚽️Activities•🚘🌇Travel•💡🎉Objects•💖🔣Symbols•🎌🏳️‍🌈Flags☺️☹☝️✌️✍️❤️❣️☠♨️✈️⌛⌚♈♉♊♋♌♍♎♏♐♑♒♓☀️☁️☂️❄️⛄️☄♠️♥️♦️♣️▶️◀️☎️⌨✉️✏️✒️✂️↗️➡️↘️↙️↖️↕️↔️↩️↪️✡️☸☯️✝️☦☪☮☢☣☑️✔️✖️✳️✴️❇️‼️©️®️™️Ⓜ️▪️▫️#⃣️*️⃣0⃣️1⃣️2⃣️3⃣️4⃣️5⃣️6⃣️7⃣️8⃣️9⃣️⁉️ℹ️⤴️⤵️♻️◻️◼️◽◾☕⚠️☔⏏⬆️⬇️⬅️⚡☘⚓♿⚒⚙⚗⚖⚔⚰⚱⚜⚛⚪⚫🀄⭐⬛⬜⛑⛰⛪⛲⛺⛽⛵⛴⛔⛅⛈⛱⛄⚽⚾️⛳⛸⛷⛹⛏⛓⛩⭕❗🅿️❦♕♛♔♖♜☾→⇒⟹⇨⇰➩➪➫➬➭➮➯➲➳➵➸➻➺➼➽☜☟➹➷↶↷✆⌘⎋⏎⏏⎈⎌⍟❥ツღ☻`
                    },
                ],
                emojiPalette: ``,
            }
        },
        mounted() {
            this.emojiPalette = this.paletteOptions[0].emojis;// Default
            this.updateScaleFactor();
            window.addEventListener('resize', () => this.updateScaleFactor());
            // this.updatePalette();

            this.renderView = new PIXI.Application({backgroundColor: 0x222222});
            this.renderView.renderer.resize(0, 0);
            document.querySelector('.render-output').appendChild(this.renderView.view);
        },
        methods: {
            updatePalette() {
                this.loading = true;
                if (this.emojiMap === null || this.emojiMap.palette !== this.emojiPalette) {

                    this.emojiMap = EmojiMapManager.emojiMap(this.emojiPalette);
                    console.log("XHERE");
                    this.updateEmojiTextureMap();
                }
                this.loading = false;
            },
            updateEmojiTextureMap() {
                console.log("Updating emoji texture map");
                const emojiSize = document.querySelector('#app').offsetWidth / this.emojiWidth;
                let emojis = EmojiSplitter.spliddit(this.emojiPalette);
                for (let emoji of emojis) {
                    let text = new PIXI.Text(emoji, {fontSize: emojiSize, fill: '#ffffff'});
                    text.updateText(); // force it to render to texture inside

                    this.spriteCache[emoji] = text.texture;
                }
            },
            loadImage() {
                let now = performance.now();
                this.loading = true;
                console.log('loading true');

                this.updatePalette();

                let file = document.querySelector('input[type=file]').files[0];
                let image = document.querySelector('.preview-image');
                let canvas = document.createElement('canvas');
                let context = canvas.getContext('2d');
                image.src = URL.createObjectURL(file);
                image.onload = () => {
                    let {width, height} = image;
                    let resizeFactor = this.emojiWidth / width;
                    width = Math.round(width * resizeFactor);
                    height = Math.round(height * resizeFactor);
                    canvas.width = width;
                    canvas.height = height;
                    canvas.style.width = '400px';
                    canvas.style.height = 'auto';
                    context.drawImage(image, 0, 0, width, height);

                    this.grid = this.createEmojiGrid(width, height);
                    let imageData = context.getImageData(0, 0, canvas.width, canvas.height).data;
                    this.pictureToEmoji(this.grid, imageData);
                    this.renderEmojiGrid(this.grid);

                    this.loading = false;
                    console.log('loading false');
                    let time = performance.now() - now;
                    console.log(`${time} ms`)
                }
            },

            renderEmojiGrid(emojiGrid) {
                if (this.useWebgl) {
                    const appWidth = document.querySelector('#app').offsetWidth;
                    const emojiSize = appWidth / this.emojiWidth;

                    this.renderView.renderer.resize(appWidth, emojiGrid.height * emojiSize);
                    // Clear stage
                    while (this.renderView.stage.children[0]) {
                        this.renderView.stage.removeChild(this.renderView.stage.children[0]);
                    }


                    for (let y = 0; y < emojiGrid.height; y++) {
                        for (let x = 0; x < emojiGrid.width; x++) {
                            let emoji = emojiGrid.grid[y][x];
                            let sprite = new PIXI.Sprite(this.spriteCache[emoji]);
                            sprite.x = x * emojiSize;
                            sprite.y = y * emojiSize;

                            this.renderView.stage.addChild(sprite);
                        }
                    }
                } else {
                    this.updateScaleFactor();
                    let html = '';
                    let outputElement = document.querySelector('.output');
                    //Make fresh html grid
                    for (let y = 0; y < emojiGrid.height; y++) {
                        html += `<div class='row'>`;
                        for (let x = 0; x < emojiGrid.width; x++) {
                            html += `<div class='emoji'>${emojiGrid.grid[y][x]}</div>`;
                        }
                        html += '</div>';
                    }
                    outputElement.innerHTML = html;
                }
            },

            createEmojiGrid(width, height) {
                let grid = [];
                for (let y = 0; y < height; y++)
                    grid.push([]);
                return {width, height, grid};
            },

            pictureToEmoji(emojiGrid, imageData) {
                for (let i = 0; i < imageData.length; i += 4) {
                    let n = i / 4;
                    let y = Math.floor(n / emojiGrid.width);
                    let x = n % emojiGrid.width;

                    let [r, g, b] = imageData.slice(i, i + 3);
                    let lab = LabConvert.rgbToLab(r, g, b);
                    for (let j = 0; j < lab.length; j++) {
                        lab[j] = Math.round(
                            (this.emojiMap.colorToEmoji.colorRange + lab[j]) /
                            this.emojiMap.colorToEmoji.step)
                    }
                    emojiGrid.grid[y][x] = this.emojiMap.colorToEmoji.map[lab[0]][lab[1]][lab[2]];
                }
            },
            updateScaleFactor() {
                if (this.useWebgl) {
                    this.updateEmojiTextureMap();
                    return;
                }
                let output = document.querySelector('.output');
                let appWidth = document.querySelector('#app').offsetWidth;
                let scaleFactor = appWidth / this.emojiWidth;
                output.style.width = this.emojiWidth + 'px';

                let newHeight;
                if (this.grid !== null)
                    newHeight = this.grid.height * scaleFactor;
                else
                    newHeight = 0;
                output.style.height = newHeight + 'px';

                output.style.transform = `scale(${scaleFactor})`;
                console.log("Scaling output with factor", scaleFactor);
            }

        },
        watch: {
            emojiWidth() {
                this.timeouts.forEach(t => clearTimeout(t));
                this.timeouts.push(setTimeout(() => {
                    this.updateScaleFactor();
                }, 500));
                // console.log("Changing emoji size to", newSize);
                // document.documentElement.style.setProperty('--emoji-size', newSize + 'px');
            }
        }
    }
</script>

<style>
    :root {
        --emoji-size: 1px;
    }

    #app {
        max-width: 800px;
        margin: 0 auto;
        display: block;
    }

    .render-output{
        margin-top:30px;
        margin-left: -20px;
        width:calc(100% + 40px);
    }

    .output {
        text-align: center;
        margin-top: 0px;
        margin-bottom: 0px;
        margin-left: -20px;
        line-height: var(--emoji-size);
        white-space: nowrap;
        font-variant-numeric: tabular-nums;
        pointer-events: none;
        transform-origin: top left;
    }

    .row {
        margin: 0 !important;
        padding: 0 !important;
        height: var(--emoji-size);
    }

    .emoji {
        background-color: rgba(0, 0, 0, 0.95);
        margin: 0 !important;
        padding: 0 !important;
        display: inline-block;
        margin-right: -4px;
        width: var(--emoji-size);
        height: var(--emoji-size);
        font-size: var(--emoji-size);
        text-align: center;
    }

    .md-content {
        padding: 20px;
    }

    html, body {
        margin: 0 !important;
        padding: 0;
    }
</style>
