<!--  -->
<template>
    <div class='payment-civil'>
        <div class='payment-civil-title'>
            <span class='payment-civil-title-go1'>我的审批</span>
            <span class='payment-civil-title-separator'> / </span>
            <span class='payment-civil-title-go2'>债权处理信息缴费审批</span>
        </div>
        <div class='payment-civil-content'>
            <div>汇款账户：</div>
            <div><input type="text" v-model='PamentMsg.CardNum' disabled='true'></div>
            <div><input type="text" v-model='PamentMsg.AccountName' disabled='true'></div>
            <div><input type="text" v-model='PamentMsg.OpeningBank' disabled='true'></div>
            <div><input type="text" v-model='PamentMsg.FeePayable' disabled='true'></div>
            <div class='payment-civil-content-update'>
                上传凭证：
                <div class='payment-civil-content-update-box'>
                    <div class='payment-civil-content-update-box-container'>
                        <img src="@imgs/home/baidu.png" alt="">
                    </div>
                    <div class='payment-civil-content-update-box-container'>
                        <img src="@imgs/home/baidu.png" alt="">
                    </div>
                    <div class='payment-civil-content-update-box-container'>
                        <img src="@imgs/home/baidu.png" alt="">
                    </div>
                    <div class='payment-civil-content-update-box-container'>
                        <img src="@imgs/home/baidu.png" alt="">
                    </div>
                </div>
                <button class='payment-civil-content-update-button'>点击上传</button>
            </div>
            <div class='payment-civil-content-payer'>
                合同人姓名：
                <input type="text" v-model='SubmitData.contractName' :disabled='true'>
            </div>
            <div class='payment-civil-content-payer'>打款人姓名：
                <input type="text" :disabled='true' v-model='SubmitData.payertName'>
            </div>
        </div>
        <div class='payment-civil-check'>
            <div class='payment-civil-check-reason'>
                <span>审批原因</span>
                <textarea maxlength='141' v-model='SubmitApproveData.checkReason'></textarea>
            </div>
            <div class='payment-civil-check-button'>
                <button @click='RejectCheck' >审核驳回</button>
                <button @click='PassCheck' >审核通过</button>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    data () {
        return {
            PamentMsg: {
                CardNum: '收款卡号：810101201421046328',
                AccountName: '开户名：山东盛世天泽公关顾问有限公司',
                OpeningBank: '开户行：日照银行股份有限公司银海支行',
                FeePayable: '应缴费用：880（元）',
                Contractor: ''
            },
            SubmitData: {},
            SubmitApproveData: {
                checkReason: '',
                status: '',
                debtId: ''
            },
            // 新增资产信息所需参数
            AddPropertyMsg: {
                relativePerId: '',
                reportId: '',
                civilId: '',
                debtId: '',
                createId: '',
                status: '0'
            },
            // 更新支付明细状态接口
            UpdatePay: {
                status: '',
                payId: ''
            },
            HasSubmit: false
        }
    },
    methods: {
        async InitData () {
            const formData = new FormData()
            formData.append('payId', this.$store.state.payId)
            const { data: result } = await this.$http({
                method: 'post',
                url: '/api/api/busPayDetailController/selectByPrimaryKey',
                data: formData,
                headers: {
                    'Content-Type': 'multipart/form-data'
                }
            })
            this.SubmitData = result.data
            console.log(this.$store.state.debtId)
        },
        async RejectCheck () {
            if (!this.SubmitApproveData.checkReason) return this.$message.error('请先填写审核原因')
            await this.UpdatePayStatus('1')
            await this.UpdateCheckStatus('8')
        },
        async PassCheck () {
            await this.UpdatePayStatus('2')
            await this.UpdateCheckStatus('9').then(() => {
                // 通过解债ID查询信息
                const formData = new FormData()
                const debtId = this.$store.state.debtId
                console.log(this.$store.state.debtId)
                formData.append('debtId', debtId)
                this.$http({
                    method: 'post',
                    url: '/api/api/pubDebtController/selectByPrimaryKey',
                    data: formData,
                    headers: {
                        'Content-Type': 'multipart/form-data'
                    }
                }).then(result => {
                    console.log(result.data)
                    this.AddPropertyMsg.relativePerId = result.data.data.relativePerId
                    this.AddPropertyMsg.reportId = result.data.data.reportId
                    this.AddPropertyMsg.civilId = result.data.data.civilId
                    this.AddPropertyMsg.debtId = this.$store.state.debtId
                    this.AddPropertyMsg.createId = window.sessionStorage.getItem('userId')
                    console.log(this.AddPropertyMsg)
                    const formData = new FormData()
                    for (const key in this.AddPropertyMsg) {
                        formData.append(key, this.AddPropertyMsg[key])
                    }
                    this.$http({
                        method: 'post',
                        url: '/api/api/busPropertController/insertSelective',
                        data: formData,
                        headers: {
                            'Content-Type': 'multipart/form-data'
                        }
                    }).then(AddResult => {
                        if (AddResult.data.resultCode !== '200') this.$message.error('新增资产信息错误')
                        // 调用stage改变接口
                        const StageUpdateformData = new FormData()
                        StageUpdateformData.append('reportId', this.$store.state.reportId)
                        StageUpdateformData.append('stage', '4')
                        this.$http({
                            method: 'post',
                            url: '/api/api/busReportController/updateDebtStage',
                            data: StageUpdateformData,
                            headers: {
                                'Content-Type': 'multipart/form-data'
                            }
                        }).then(StageUpdateResult => {
                            if (StageUpdateResult.data.resultCode !== '200') return this.$message.error(StageUpdateResult.data.resultMessage)
                            this.$message.success('进入资产阶段，新增资产信息成功')
                            this.HasSubmit = true
                        })
                        // 调用报备状态更改的接口
                    })
                })
            })
        },
        // 提交审批状态
        async UpdateCheckStatus (status) {
            const formData = new FormData()
            this.SubmitApproveData.status = status
            this.SubmitApproveData.debtId = this.$store.state.debtId
            for (const key in this.SubmitApproveData) {
                formData.append(key, this.SubmitApproveData[key])
            }
            const { data: result } = await this.$http({
                method: 'post',
                url: '/api/api/pubDebtController/updateStatus',
                data: formData,
                headers: {
                    'Content-Type': 'multipart/form-data'
                }
            })
            if (result.data !== '0' && result.resultCode === '200') {
                this.$message.success(result.resultMessage)
            } else {
                this.$message.error('操作失败, 请重试')
            }
            return result
        },
        // 更改
        UpdatePayStatus (status) {
            const formData = new FormData()
            this.UpdatePay.status = status
            this.UpdatePay.payId = this.$store.state.payId
            for (const key in this.UpdatePay) {
                formData.append(key, this.UpdatePay[key])
            }
            this.$http({
                method: 'post',
                url: '/api/api/busPayDetailController/updateStatus',
                data: formData,
                headers: {
                    'Content-Type': 'multipart/form-data'
                }
            }).then(result => {
                if (result.resultCode !== '200') return this.$message.error(result.resultMessage)
            })
        }
    },
    created () {
        this.InitData()
    }
}

</script>
<style lang='scss' scoped>
@import '@css/style.scss';
.payment-civil {
    display: flex;
    flex-direction: column;
    background-color: #E9F0F5;
    height: 100%;
    width: 100%;
    box-sizing: border-box;
    padding: 0 px2rem(4);
    &-title {
        height: px2rem(12);
        line-height: px2rem(12);
        font-size: px2rem(4);
        color: #FC7F89;
        &-go1 {
            color: #616789;
        }
        &-separator {
            color: #616789
        }
    }
    &-content {
        background-color: #fff;
        border-radius: px2rem(2);
        height: 100%;
        font-size: px2rem(3.5);
        padding: px2rem(4);
        box-sizing: border-box;
        line-height: px2rem(8);
        display: flex;
        flex-direction: column;
        align-items: flex-start;

        input {
            width: px2rem(115);
            padding-left: px2rem(5);
            background: #F0F2FD;
            border: none;
            height: px2rem(10);
            line-height: px2rem(10);
            margin: px2rem(1) 0;
            border-radius: px2rem(1);
        }
        &-title {
            font-weight: 600;
        }
        &-update {
            height: px2rem(16);
            display: flex;
            margin: px2rem(4) 0;
            &-box {
                width: px2rem(140);
                height: px2rem(16);
                border: 1px solid #E8EAEC;
                margin: 0 px2rem(4);
                display: flex;
                align-items: center;

                &-container {
                    border: 1px solid #E8E8E8;
                    display: flex;
                    justify-content: center;
                    align-items: center;
                    margin: 0 px2rem(1);
                    height: px2rem(10);
                    width: px2rem(14);
                    img {
                        width: px2rem(14);
                        height: px2rem(8)
                    }
                }
            }
            &-button {
                height: px2rem(9);
                width: px2rem(25);
                border: none;
                background-color: #616789;
                color: #fff;
                border-radius: px2rem(2);
            }
        }
        &-payer {
            margin-top: px2rem(2);
            input {
                width: px2rem(80);
                height: px2rem(8);
                background-color: #fff;
                border: 1px solid #E8EAEC;
            }
            input::-webkit-input-placeholder{
                font-size: px2rem(3.5);
            }
            input::-moz-placeholder{   /* Mozilla Firefox 19+ */
                font-size: px2rem(3.5);
            }
            input:-moz-placeholder{    /* Mozilla Firefox 4 to 18 */
                font-size: px2rem(3.5);
            }
            input:-ms-input-placeholder{  /* Internet Explorer 10-11 */
                font-size: px2rem(3.5);
            }
        }
        &-submit {
            width: px2rem(60);
            height: px2rem(10);
            background-color: #616789;
            color: #fff;
            border: none;
            border-radius: px2rem(1);
            margin-top: px2rem(4);
        }
    }
    &-check {
        align-items: center;
        height: px2rem(60);
        background-color: #fff;
        width: px2rem(160);
        margin: px2rem(4) auto;
        border-radius: px2rem(2);
        &-reason {
            margin: px2rem(10) 0;
            display: flex;
            justify-content: center;
            align-items: center;
            span {
                font-size: px2rem(5);
                margin-right: px2rem(4);
            }
            textarea {
                width: px2rem(100);
                height: px2rem(20);
                border-radius: px2rem(1);
                font-size: px2rem(4);
                resize: none;
                border: 1px solid #E0E3F8;
                line-height: px2rem(6);
            }
        }
        &-button {
            display: flex;
            justify-content: center;
            align-items: center;
            button {
                margin: 0 px2rem(4);
                padding: px2rem(1.4) px2rem(4);
                height: px2rem(10);
                font-size: px2rem(3.2);
                border-radius: px2rem(1);
                color: #fff;
                border: none;
                font-size: px2rem(4);
            }
            :nth-child(1) {
                background-color: #616789;
            }
            :nth-child(2) {
                background-color: #FC7F89;
            }
        }
    }
}
</style>
