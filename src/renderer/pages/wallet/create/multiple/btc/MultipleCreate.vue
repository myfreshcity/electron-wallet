<template>
  <div class="wallet-multiple-btc-create">
    <Header
        v-show="!showDialog"
        headerTitle="决策权分配"
    ></Header>
    <div class="wallet-multiple-btc-create-items">
      <div class="wallet-multiple-btc-create-item">
        <p class="wallet-multiple-btc-create-main">机构名称</p>
        <div class="wallet-multiple-btc-create-sub">
          <input class="wallet-multiple-btc-create-text" v-model="project" type="text" readonly="readonly">
        </div>
      </div>
      <div class="wallet-multiple-btc-create-item">
        <p class="wallet-multiple-btc-create-main">多签钱包别名</p>
        <div class="wallet-multiple-btc-create-sub">
          <input class="wallet-multiple-btc-create-text" v-model="alias" type="text" placeholder="请输入钱包别名">
        </div>
      </div>
    </div>
    <div class="wallet-multiple-btc-create-members">
      <div class="title">
        <img class="right" src="../../../../../../../static/images/common/increase_1.png" @click="addSelf"/>
        <p class="left">添加个人账户</p>
      </div>
      <div class="content" v-if="selfList && selfList.length>0">
        <div class="members">
          <div class="members-right">
            <button @click="removeSelf">删除</button>
          </div>
          <div class="members-left">
            <div class="one">
              <p class="member-alias">{{ selfList[0].walletAlias }}</p>
              <p class="member-username">{{ selfList[0].userName }}</p>
            </div>
            <p class="two">{{ selfList[0].userEmail }}</p>
            <p class="three">{{ selfList[0].walletAddress }}</p>
          </div>
        </div>
      </div>
    </div>
    <div class="wallet-multiple-btc-create-members">
      <div class="title">
        <img class="right" src="../../../../../../../static/images/common/increase_1.png" @click="addContact"/>
        <p class="left">添加参与者账户</p>
      </div>
      <div class="content" v-if="contactList && contactList[key]" v-for="(item, key) in contactList">
        <div class="members">
          <div class="members-right">
            <button @click="removeContact(item.walletAddress)">删除</button>
          </div>
          <div class="members-left">
            <div class="one">
              <p class="member-alias">{{ item.walletAlias }}</p>
              <p class="member-username">{{ item.userName }}</p>
            </div>
            <p class="two">{{ item.userEmail }}</p>
            <p class="three">{{ item.walletAddress }}</p>
          </div>
        </div>
      </div>
    </div>
    <div class="wallet-multiple-btc-create-members">
      <div class="title">
        <img class="right" v-if="bossList && bossList.length>0" src="../../../../../../../static/images/common/increase_1.png" @click="addBoss"/>
        <img class="right" v-else src="../../../../../../../static/images/common/increase_2.png"/>
        <p class="left">必要决策人</p>
      </div>
      <div class="content" v-if="boss">
        <div class="members">
          <div class="members-right">
            <button @click="removeBoss">删除</button>
          </div>
          <div class="members-left">
            <div class="one">
              <p class="member-alias">{{ boss.walletAlias }}</p>
              <p class="member-username">{{ boss.userName }}</p>
            </div>
            <p class="two">{{ boss.userEmail }}</p>
            <p class="three">{{ boss.walletAddress }}</p>
          </div>
        </div>
      </div>
    </div>
    <div class="multiple-btc-count">
      <p class="multiple-btc-count-p">需要签名人数</p>
      <div class="multiple-btc-count-item">
        <img class="left" src="../../../../../../../static/images/common/minus.png" @click="min">
        <img class="right" src="../../../../../../../static/images/common/add.png" @click="plus">
        <div class="middle">
          <input class="multiple-btc-count-input" v-model="confirmNum"/>
        </div>
      </div>
    </div>
    <Contact
        v-if="showDialog && listType===0"
        :listType="listType"
        selectType="single"
        :coin="coin"
        @dialogChecked="dialogChecked"
    ></Contact>
    <Contact
        v-if="showDialog && listType===1"
        :listType="listType"
        selectType="multi"
        :coin="coin"
        @dialogChecked="dialogChecked"
    ></Contact>
    <Contact
        v-if="showDialog && listType===2"
        :listType="listType"
        selectType="single"
        :list="bossList"
        @dialogChecked="dialogChecked"
    ></Contact>
    <div class="multiple-btc-btns">
      <button class="create-btn" @click="create" :disabled="isLoading">创建</button>
    </div>
    <ball-beat-loader v-if="isLoading" color="#3B78C3"></ball-beat-loader>
  </div>
</template>

<script>
  import Header from '@/common/Header';
  import bitcoin from 'bitcoinjs-lib';
  import Store from 'electron-store';
  import Wallet from '@/api/Wallet';
  import Dict from '@/config/dict';
  import Base from '@/config/base';
  import Contact from '../Contact';

  export default {
    components: {
      Header,
      Contact,
    },
    data() {
      return {
        project: this.$store.state.fund.projectname,
        alias: '',
        showDialog: false,
        listType: 0, // 0：个人账户 1：参与者账户 2：必要决策人
        selfList: [],
        contactList: [],
        bossList: [],
        boss: null,
        confirmNum: 1,
        storeData: null,
        maxContact: 5, // 多签参与者最多几位
        isLoading: false,
        coin: Dict.coin.BTC,
      };
    },
    methods: {
      plus() {
        if (this.confirmNum < this.maxContact) {
          this.confirmNum = this.confirmNum + 1;
        }
      },
      min() {
        if (this.confirmNum > 1) {
          this.confirmNum = this.confirmNum - 1;
        }
      },
      addSelf() {
        this.listType = 0;
        this.selfList = [];
        this.showDialog = true;
      },
      addContact() {
        this.listType = 1;
        this.contactList = [];
        this.showDialog = true;
      },
      addBoss() {
        this.listType = 2;
        this.showDialog = true;
      },
      dialogChecked(data) {
        this.showDialog = false;
        if (data.length > 0) {
          switch (this.listType) {
            case 0:
              data.forEach((item) => {
                this.selfList.push(item);
              });
              this.updateBosslist();
              break;
            case 1:
              data.forEach((item) => {
                this.contactList.push(item);
              });
              this.updateBosslist();
              break;
            case 2:
              data.forEach((item) => {
                this.boss = item;
              });
              break;
            default:
              break;
          }
        }
      },
      updateBosslist() {
        this.boss = null;
        this.bossList = [];
        this.selfList.forEach((item) => {
          this.bossList.push(item);
        });
        this.contactList.forEach((item) => {
          this.bossList.push(item);
        });
      },
      removeSelf() {
        this.selfList = [];
        this.updateBosslist();
      },
      removeContact(address) {
        this.contactList.forEach((item, key) => {
          if (item.walletAddress === address) {
            delete this.contactList[key];
            this.$forceUpdate();
          }
        });
        this.updateBosslist();
      },
      removeBoss() {
        this.boss = null;
      },
      create() {
        this.isLoading = true;
        if (!this.alias) {
          this.notify('请输入钱包别名');
        } else if (this.confirmNum > this.bossList.length) {
          this.notify('指定人数不大于选择人数');
        } else if (this.confirmNum === 1) {
          this.notify('需要签名人数需大于1');
        } else {
          const keys = [];
          this.bossList.forEach((item) => {
            keys.push(item.publicKey);
          });
          const pubkeys = keys.map(this.filter);
          const { address } = bitcoin.payments.p2sh({
            redeem: bitcoin.payments.p2ms({ m: this.confirmNum, pubkeys }),
          });
          this.storeData = {
            [address]: {
              self: this.selfList,
              other: this.contactList,
              confirmNum: this.confirmNum,
              alias: this.alias,
            },
          };
          // save address
          const store = new Store();
          const walletList = `${Base.walletList}_${this.$store.state.user.id}_BTC`;
          const old = store.get(walletList);
          if (typeof old === 'undefined') {
            store.set(walletList, this.storeData);
          } else if (Object.prototype.hasOwnProperty.call(old, address) === false) {
            const copy = Object.assign(old, this.storeData);
            store.set(walletList, copy);
          }
          let contacts = `[${this.selfList[0].walletAddress},`;
          Object.keys(this.contactList).forEach((item) => {
            contacts += `${this.contactList[item].walletAddress},`;
          });
          contacts = contacts.substring(0, (contacts.length - 1));
          contacts += ']';
          // save address to back
          Wallet.saveMultWallet({
            walletAlias: this.alias,
            walletAddress: address,
            userEmail: this.$store.state.user.email,
            projectId: this.$store.state.fund.projectid,
            status: 1,
            type: 2, // 1 单签 2 多签
            contactList: contacts,
            passNum: this.confirmNum,
            userId: this.$store.state.user.id,
            walletSource: 1, // 1 自我创建 2 相继导入 3 参与多签
            publicKey: '',
            currencyType: '00',
            signAddress: this.selfList[0].walletAddress,
            mustSignAddressList: `[${this.boss.walletAddress}]`,
          }).then(this.saveSucc).catch(this.saveErr);
        }
      },
      filter(hex) {
        return Buffer.from(hex, 'hex');
      },
      saveSucc(res) {
        if (res.resultCode === '200') {
          this.$router.push('/wallet');
          this.notify('创建成功');
        } else {
          this.notify(res.resultMsg);
        }
      },
      saveErr() {
        this.notify('创建失败');
      },
      notify(message) {
        this.$toast(message, {
          horizontalPosition: 'center',
          verticalPosition: 'center',
          className: 'wallet-toast',
        });
        this.isLoading = false;
      },
    },
    mounted() {
    },
  };
</script>

<style lang="less" scoped>
  @import '~@/assets/styles/varibles.less';
  .wallet-multiple-btc-create {
    .wallet-multiple-btc-create-items {
      padding: 46px 0 30px 0;
      border-bottom: 4px solid #EFEFF5;
      .wallet-multiple-btc-create-item {
        margin: 0 30px;
        height: 120px;
        .wallet-multiple-btc-create-main {
          padding: 0 30px;
          color: @darkColor;
          font-size: @fontSizeFive;
          height: 72px;
          line-height: 72px;
        }
        .wallet-multiple-btc-create-sub {
          padding: 0 30px;
          height: 48px;
          line-height: 48px;
          border-bottom: 1px solid @borderBottomColor;
          .wallet-multiple-btc-create-text {
            height: 30px;
            line-height: 30px;
            width: 100%;
            font-size: @fontSizeEight;
            color: @greyColor;
          }
        }
      }
    }
    .wallet-multiple-btc-create-members {
      padding: 26px 60px 30px 60px;
      border-bottom: 4px solid #EFEFF5;
      .title {
        line-height: 46px;
        height: 46px;
        .right {
          float: right;
          height: 46px;
          width: 46px;
        }
        .left {
          margin-right: 46px;
          line-height: 46px;
          height: 46px;
          font-size: @fontSizeFive;
          color: @darkColor;
        }
      }
      .content {
        margin-top: 30px;
        .members {
          padding-bottom: 16px;
          clear: both;
          .members-right {
            float: right;
            height: 150px;
            line-height: 150px;
            width: 90px;
            border-left: 1px solid @borderBottomColor;
            text-align: center;
            color: #FF2D55;
            font-size: @fontSizeNine;
          }
          .members-left {
            margin-right: 90px;
            padding: 0 30px;
            color: @greyColor;
            .one {
              line-height: 50px;
              font-size: @fontSizeSix;
              .member-alias {
                float: left;
                padding-right: 34px;
              }
              .member-username {
                flex: 1;
              }
            }
            .two {
              line-height: 50px;
              font-size: @fontSizeSix;
            }
            .three {
              font-size: @fontSizeNine;
            }
          }
        }
      }
    }
    .multiple-btc-count {
      padding: 0 30px;
      .multiple-btc-count-p {
        margin: 30px 30px 16px 30px;
        height: 46px;
        line-height: 46px;
        color: @darkColor;
        font-size: @fontSizeFive;
      }
      .multiple-btc-count-item {
        margin-left: 30px;
        height: 60px;
        width: 226px;
        border: 1px solid #8FA1B3;
        border-radius: 30px;
        .left {
          padding: 16px;
          float: left;
          width: 62px;
        }
        .right {
          padding: 16px;
          float: right;
          width: 62px;
        }
        .middle {
          margin: 0 62px;
          height: 60px;
          line-height: 60px;
          .multiple-btc-count-input {
            text-align: center;
            height: 60px;
            line-height: 60px;
            width: 100%;
            font-size: @fontSizeFive;
          }
        }
      }
    }
    .multiple-btc-btns {
      padding: 80px 30px 60px 30px;
      .create-btn {
        height: 70px;
        width: 100%;
        background: @warningColor;
        color: #FFF;
        font-size: @fontSizeFive;
        border-radius: 35px;
      }
    }
  }
</style>

<style lang="less">
  .wallet-multiple-btc-create {
    .v-list__tile {
      height: 120px;
    }
  }
</style>
