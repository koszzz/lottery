<script>
import { ElMessage, ElNotification, dayjs } from 'element-plus'
import customParseFormat from 'dayjs/plugin/customParseFormat'
dayjs.extend(customParseFormat)

export default {
    data() {
        return {
            replies: [],
            finalIndex: 0,
            step: 0,
            duplicates: -1,
            seed: dayjs().unix(),
            randomNum: 5,
            randomNumbers: [],
            stopSeed: false
        };
    },
    created() {
        fetch('data.json').then(res => res.json()).then(data => {
            this.replies = data.sort(function (a, b) {
                return a.sequence - b.sequence;
            });
            ElMessage({
                message: `成功获取到${data.length}条回复`,
                type: 'success',
            })
        })
        setInterval(() => {
            if (!this.stopSeed) {
                this.seed = dayjs().unix()
            }
        }, 1000)
    },
    methods: {
        getTime(time) {
            return dayjs.unix(+time).format('YYYY/MM/DD HH:mm:ss')
        },
        removeDuplicates() {
            const arr = this.replies
            let unique = {};
            let duplicates = [];
            arr.forEach(function (item) {
                if (!unique[item.user.id]) {
                    unique[item.user.id] = item;
                } else {
                    duplicates.push(item);
                    duplicates.push(unique[item.user.id]);
                    delete unique[item.user.id];
                }
            });
            duplicates.reverse()
            const groupByUserId = (arr) => {
                const grouped = arr.reduce((acc, item) => {
                    if (!acc[item.user.id]) {
                        acc[item.user.id] = [];
                    }
                    acc[item.user.id].push(item);
                    return acc;
                }, {});
                return Object.values(grouped).flat();
            }
            this.duplicates = groupByUserId(duplicates)
            const beforeLength = this.replies.length
            this.replies = Object.values(unique).sort(function (a, b) {
                return a.sequence - b.sequence;
            });
            ElNotification({
                title: '去重完成',
                message: `${beforeLength}条回复中，${beforeLength - this.replies.length}条重复！`,
                type: 'success',
            })
            return beforeLength;
        },
        nextStep() {
            this.step += 1
        },
        random() {
            this.stopSeed = true
            let seed = parseInt(this.seed)
            this.seed = seed
            const count = this.randomNum
            const min = 0
            const max = this.replies.length - 1
            // 创建一个数组来存储生成的随机数
            let result = [];
            // 使用种子初始化随机数生成器
            let rng = function () {
                seed = (seed * 9301 + 49297) % 233280;
                return seed / 233280;
            };
            // 循环直到生成足够数量的随机数
            while (result.length < count) {
                // 生成一个随机数
                let num = Math.floor(rng() * (max - min + 1)) + min;
                // 检查是否已经在结果数组中
                if (result.indexOf(num) === -1) {
                    result.push(num);
                }
            }
            this.randomNumbers = result
        },
        getRes() {
            let res=[]
            const randomNumbers = this.randomNumbers
            for (let i = 0; i < randomNumbers.length; i++) {
                this.replies[randomNumbers[i]].index = randomNumbers[i]
                res.push(this.replies[randomNumbers[i]]);
            }
            this.step += 1
            return res
        },
        seedFocus(){
            this.stopSeed = true
        }
    },
    computed: {
    }
}
</script>

<template>
    <div class="common-layout">
        <el-container>
            <el-container>
                <el-main style="display: flex;justify-content: space-between;">
                    <div class="leftArea">
                        <el-table :data="replies" border style="width: 100%;height:720px;">
                            <el-table-column type="index" :index="(index) => index" label="索引" width="53" />
                            <el-table-column prop="sequence" label="楼层" width="53" />
                            <el-table-column prop="user.name" label="昵称" width="180" />
                            <el-table-column label="回复时间" width="180">
                                <template #default="scope">
                                    <span>{{ getTime(scope.row.createTime) }}</span>
                                </template>
                            </el-table-column>
                            <el-table-column prop="user.id" label="用户id" />
                        </el-table>
                    </div>
                    <div class="rightArea">
                        <el-steps :active="step" finish-status="success" simple style="">
                            <el-step title="去重" />
                            <el-step title="随机" />
                            <el-step title="完成" />
                        </el-steps>
                        <div class="rightContainer">
                            <div v-if="step == 0">
                                <el-button type="primary" @click="removeDuplicates" size="large"
                                    :disabled="duplicates != -1">去重</el-button>
                                <h3>以下是重复项</h3>
                                <el-table :data="duplicates == -1 ? [] : duplicates" border
                                    style="width: 100%;height: 552px">
                                    <el-table-column prop="sequence" label="楼层" width="53" />
                                    <el-table-column prop="user.name" label="昵称" width="180" />
                                    <el-table-column label="回复时间" width="180">
                                        <template #default="scope">
                                            <span>{{ getTime(scope.row.createTime) }}</span>
                                        </template>
                                    </el-table-column>
                                    <el-table-column prop="user.id" label="用户id" />
                                </el-table>
                                <el-button type="success" @click="nextStep" style="margin-top: 10px;"
                                    :disabled="duplicates == -1">下一步</el-button>
                            </div>
                            <div v-if="step == 1">
                                <div style="display: flex;align-items: center;">
                                    <h3 style="width: 180px;">
                                        当前随机数种子：
                                    </h3>
                                    <el-input-number v-model="seed" size="large" :disabled="randomNumbers.length > 0" @focus="seedFocus" />
                                </div>
                                <div style="display: flex;align-items: center;">
                                    <h3>
                                        抽取
                                    </h3>
                                    <el-input-number v-model="randomNum" style="margin-left: 5px;margin-right: 5px;"
                                        :disabled="randomNumbers.length > 0" />
                                    <h3>
                                        个随机数索引
                                    </h3>
                                </div>
                                <el-button type="primary" @click="random" size="large"
                                    :disabled="randomNumbers.length > 0">抽取索引</el-button>
                                <div style="display: flex;">
                                    <div class="randomNum" v-for="num in randomNumbers">{{ num }}</div>
                                </div>
                                <el-button type="success" @click="nextStep" style="margin-top: 10px;"
                                    :disabled="randomNumbers.length == 0">下一步</el-button>
                            </div>
                            <div v-if="step >= 2">
                                <el-table :data="getRes()" border
                                    style="width: 100%;height: 655px">
                                    <el-table-column prop="index" label="索引" width="53" />
                                    <el-table-column prop="sequence" label="楼层" width="53" />
                                    <el-table-column label="头像" width="125" >
                                        <template #default="scope">
                                            <img :src="scope.row.user.icon" :alt="scope.row.user.name+'的头像'" width="100" height="100" />
                                        </template>
                                        </el-table-column>
                                    <el-table-column prop="user.name" label="昵称" width="180" />
                                    <el-table-column label="回复时间" width="180">
                                        <template #default="scope">
                                            <span>{{ getTime(scope.row.createTime) }}</span>
                                        </template>
                                    </el-table-column>
                                    <el-table-column prop="user.id" label="用户id" />
                                </el-table>
                            </div>
                        </div>
                    </div>
                </el-main>
                <el-footer style="border-top: 1px solid var(--border-color);display: flex;align-items: center;">
                    <el-link>
                        上善若水，版权所有。
                    </el-link>
                    <el-link href="https://beian.miit.gov.cn" target="_blank">鲁ICP备2022004418号-2</el-link>
                    <el-link target="_blank"
                        href="https://www.beian.gov.cn/portal/registerSystemInfo?recordcode=37028102001219">
                        <img src="./assets/ghs.png" width="20" height="20" alt="公安徽标">
                        <span>鲁公网安备 37028102001219号</span>
                    </el-link>
                    <el-link href="https://www.12377.cn/" target="_blank">中国互联网违法和不良信息举报中心</el-link>
                    <el-link>
                        来自青岛，用<span style="color:red;">♥</span>制作
                    </el-link>
                </el-footer>
            </el-container>
        </el-container>
    </div>
</template>

<style>
#app {
    width: 100%;
    height: 100%;
}

.common-layout,
.el-container {
    width: 100%;
    height: 100%;
}

.el-footer>a {
    margin-left: 1rem;
}

.leftArea,
.rightArea {
    height: 100%;
}

.leftArea {
    width: 33%;
}

.rightArea {
    width: 63%;
}

.rightContainer {
    margin-top: 20px;
}

.randomNum {
    width: 80px;
    height: 80px;
    border: 1px solid rgb(207, 217, 222);
    border-radius: 15px;
    margin-top: 10px;
    margin-right: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 30px;
}
</style>
