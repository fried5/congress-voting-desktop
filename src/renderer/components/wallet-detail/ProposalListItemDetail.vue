<template>
  <div :class="['ProposalListItemDetail', imminent, votingResult]">
    <div class="code">{{ item.code }}</div>
    <h2 class="title">{{ item.title }}</h2>
    <span class="time" v-if="remainTime">{{ remainTime }}</span>
    <span class="time" v-else-if="votingResult">{{ $t('this congress voting is closed') }}</span>
    <hr class="line1" v-if="closed">
    <div :class="['result', votingResult]" v-if="closed">
      {{ $t(votingResult) }}
      <span v-if="voted">{{ $t('you voted for it')}}</span>
    </div>
    <div class='chart' v-if="hasResult">
      <span class="legend">
        {{$t('congress member')}}
        <span class="value">{{congressMemberCount}}</span>
      </span>
      <span class="legend">
        {{$t('total votes')}}
        <span class="value">{{totalVoteCount}}</span>
      </span>
      <div class="chartWrap">
        <div id="resultChart"></div>
        <div class="counts">
          <div>
            <div class="agree">
              <span class="label">{{$t('agree')}}</span>
              <span class="votes">{{item.result_yes.toLocaleString()}} ({{votesPercentage(item.result_yes)}}%)</span>
            </div>
            <div class="disagree">
              <span class="label">{{$t('disagree')}}</span>
              <span class="votes">{{item.result_no.toLocaleString()}} ({{votesPercentage(item.result_no)}}%)</span>
            </div>
            <div class="abstain">
              <span class="label">{{$t('abstain')}}</span>
              <span class="votes">{{item.result_abs.toLocaleString()}} ({{votesPercentage(item.result_abs)}}%)</span>
            </div>
          </div>
        </div>
      </div>
      <div class="approvalCondition">
        {{$t('the proposal approval condition', { ratio: item.condition_ratio })}}<br>
        {{$t('the proposal will be approved if', { ratio: item.condition_ratio * 100 })}}
      </div>
    </div>
    <div class="btns" v-if="votable">
      <div v-if="voted">
        <button class="btn edit" @click="change">{{$t('change my vote')}}</button>
      </div>
      <div v-else>
        <button class="btn agree" @click="openPassphraseDialog('yes')">{{$t('agree')}}</button>
        <button class="btn disagree" @click="openPassphraseDialog('no')">{{$t('disagree')}}</button>
        <button class="btn abstain" @click="openPassphraseDialog('abs')">{{$t('abstain')}}</button>
      </div>
    </div>
    <hr class="line3">
    <div class="contract" v-html="highlightedContract"></div>
    <bos-passphrase-dialog ref="passphraseDialog"/>
    <bos-toast v-model="showMessage" :timeout="2500" pullRight>{{message}}</bos-toast>
  </div>
</template>

<script>
  import Prism from 'prismjs';
  import 'prismjs/components/prism-yaml';
  import c3 from 'c3';

  import 'prismjs/themes/prism.css';

  export default {
    name: 'bos-wallet-proposal-item-detail',
    props: ['item', 'wallet'],
    data() {
      return {
        answer: null,
        showMessage: false,
        message: '',
        voted: false,
        chart: null,
      };
    },
    methods: {
      change(evt) {
        this.voted = false;
        evt.target.blur();
      },
      openPassphraseDialog(answer) {
        this.answer = answer;
        this.$refs.passphraseDialog.open({
          confirmCallback: this.vote,
        });
      },
      async vote({ passphrase }) {
        try {
          await this.$store.dispatch('vote', {
            address: this.wallet.address,
            proposalId: this.item.id,
            answer: this.answer,
            passphrase,
          });
          this.checkVoted();
        } catch (err) {
          if (err.message.match(/bad decrypt/)) {
            this.message = this.$t('passphrase is wrong');
          } else if (err.response && err.response.status >= 400 && err.response.status < 500) {
            if (err.response.data && err.response.data.error) {
              this.message = err.response.data.error;
            } else {
              this.message = this.$t('something is wrong');
            }
          } else {
            this.message = this.$t('something is wrong');
          }

          this.showMessage = true;
        }
      },
      async checkVoted() {
        this.voted = await this.$store.getters.hasVotedForProposal(this.item.id, this.wallet.address);
      },
      votesPercentage(v) {
        if (!this.totalVoteCount) {
          return 0;
        }
        return ((parseInt(v, 10) / this.totalVoteCount) * 100).toFixed(1);
      },
    },
    computed: {
      imminent() {
        // 17280 => block for 1 day = 60 * 60 * 24 / 5
        if (this.item.state === 'opened' && this.item.remain < 17280) {
          return 'imminent';
        }
        return null;
      },
      remainTime() {
        if (this.item.remain) {
          const now = new Date();
          const until = new Date();
          until.setSeconds(until.getSeconds() + (this.item.remain * 5));
          const diff = new Date(until.getTime() - now.getTime());
          const time = `${diff.getUTCHours()}:${diff.getUTCMinutes()}:${diff.getUTCSeconds()}`;

          if (diff.getUTCDate() - 1 > 0) {
            if (this.item.state === 'opened') {
              return this.$t('voting remained more than a day', {
                day: diff.getUTCDate() - 1,
                time,
              });
            } else if (this.item.state === 'open-before') {
              return this.$t('voting start in more than a day', {
                day: diff.getUTCDate() - 1,
                time,
              });
            }
          }

          if (this.item.state === 'opened') {
            return this.$t('voting remained less than a day', {
              time,
            });
          } else if (this.item.state === 'open-before') {
            return this.$t('voting start in less than a day', {
              day: diff.getUTCDate() - 1,
              time,
            });
          }
        }
        return null;
      },
      votingResult() {
        if (this.closed) {
          if (this.item.result_final) {
            return this.item.result_final;
          }
          return 'counting';
        }
        return null;
      },
      hasResult() {
        if (this.closed && this.item.result_final) {
          return this.item.result_final;
        }
        return null;
      },
      votable() {
        return this.item.state === 'opened';
      },
      closed() {
        return this.item.state === 'closed';
      },
      highlightedContract() {
        return Prism.highlight(this.item.content, Prism.languages.yaml, 'yaml');
      },
      congressMemberCount() {
        return this.item.result_count
          ? this.item.result_count.toLocaleString()
          : null;
      },
      totalVoteCount() {
        return parseInt(this.item.result_yes, 10) +
            parseInt(this.item.result_no, 10) +
            parseInt(this.item.result_abs, 10);
      },
    },
    mounted() {
      this.$store.state.App.ga.send('screenview', { cd: 'bos-wallet-proposal-item-detail' });
      this.checkVoted();

      if (document.querySelector('#resultChart')) {
        this.chart = c3.generate({
          bindto: '#resultChart',
          size: {
            width: 400,
            height: 400,
          },
          data: {
            columns: [
              ['agree', this.item.result_yes],
              ['disagree', this.item.result_no],
              ['abstain', this.item.result_abs],
            ],
            colors: {
              agree: '#1792f0',
              disagree: '#ed6060',
              abstain: '#728395',
            },
            type: 'donut',
          },
          interaction: {
            enabled: false,
          },
          donut: {
            label: {
              show: false,
              expand: false,
            },
            width: 15,
          },
        });
        this.chart.legend.hide();
      }
    },
    destroyed() {
      this.answer = null;
      this.voted = false;
      if (this.chart) {
        this.chart.destroy();
      }
    },
  };
</script>

<style>
  .ProposalListItemDetail {
    width: 100%;
    border-radius: 2px;
    background-color: #ffffff;
    margin: 5px;
    padding: 30px 40px;
    position: relative;
  }

  .ProposalListItemDetail .code {
    font-size: 12px;
    color: #1792f0;
  }

  .ProposalListItemDetail .time {
    font-size: 11px;
    text-align: right;
    color: #728395;
    position: absolute;
    top: 30px;
    right: 40px;
  }

  .ProposalListItemDetail .time:before {
    content: "";
    width: 12px;
    height: 12px;
    display: inline-block;
    vertical-align: middle;
    margin-right: 5px;
    margin-top: -2px;
    background: url(../../assets/svg/proposal-clock.svg);
  }

  .ProposalListItemDetail.imminent .time {
    color: #ec1f1f;
  }

  .ProposalListItemDetail.imminent .time:before {
    background: url(../../assets/svg/proposal-clock-highlight.svg);
  }

  .ProposalListItemDetail .title {
    margin-top: 30px;
  }

  .ProposalListItemDetail .line1,
  .ProposalListItemDetail .line3 {
    width: 100%;
    height: 1px;
    background-color: #c4d1d6;
    margin: 40px 0;
    border: none;
  }

  .ProposalListItemDetail .contract {
    white-space: pre-wrap;
    word-wrap: break-word;
    font-family: inherit;
  }

  .ProposalListItemDetail .contract .token.number {
    color: inherit;
  }

  .ProposalListItemDetail .btns {
    text-align: center;
    margin-top: 40px;
  }

  .ProposalListItemDetail .btn {
    width: 150px;
    height: 45px;
    margin: 0 5px;
    border-radius: 2px;
    background-color: #ffffff;
    font-size: 15px;
  }

  .ProposalListItemDetail .btn.agree {
    color: #1792f0;
    border: 1px solid #1792f0;
  }

  .ProposalListItemDetail .btn.agree:hover {
    box-shadow: 0 5px 13px 0 #cde6ff;
  }

  .ProposalListItemDetail .btn.agree:active {
    border-color: #3da4f3;
    background-color: #3da4f3;
    color: #ffffff;
  }

  .ProposalListItemDetail .btn.disagree {
    color: #ed6060;
    border: 1px solid #ed6060;
  }

  .ProposalListItemDetail .btn.disagree:hover {
    box-shadow: 0 5px 13px 0 #ffd6d7;
  }

  .ProposalListItemDetail .btn.disagree:active {
    border-color: #ed6060;
    background-color: #ed6060;
    color: #ffffff;
  }

  .ProposalListItemDetail .btn.abstain,
  .ProposalListItemDetail .btn.edit {
    color: #728395;
    border: 1px solid #728395;
  }

  .ProposalListItemDetail .btn.abstain:hover,
  .ProposalListItemDetail .btn.edit:hover {
    box-shadow: 0 5px 13px 0 #d6e0ea;
  }

  .ProposalListItemDetail .btn.abstain:active,
  .ProposalListItemDetail .btn.edit:active {
    border-color: #728395;
    background-color: #728395;
    color: #ffffff;
  }

  .ProposalListItemDetail .result {
    text-transform: uppercase;
    font-size: 40px;
    font-weight: bold;
    text-align: center;
  }

  .ProposalListItemDetail .result.counting {
    color: #000000;

  }
  .ProposalListItemDetail .result.passed {
    color: #1792f0;
  }
  .ProposalListItemDetail .result.rejected {
    color: #ed6060;
  }

  .ProposalListItemDetail .result span {
    display: block;
    font-size: 10px;
    color: #728395;
  }

  .ProposalListItemDetail.counting .line1,
  .ProposalListItemDetail.passed .line1,
  .ProposalListItemDetail.rejected .line1 {
    margin-top: 40px;
    margin-bottom: 20px;
  }

  .ProposalListItemDetail.counting .line3,
  .ProposalListItemDetail.passed .line3,
  .ProposalListItemDetail.rejected .line3 {
    margin-top: 20px;
    margin-bottom: 40px;
  }

  .ProposalListItemDetail .chart {
    margin-top: 12px;
    text-align: center;
  }

  .ProposalListItemDetail .chart .legend {
    margin-left: 20px;
    margin-right: 20px;
  }

  .ProposalListItemDetail .chart .legend .value {
    margin-left: 10px;
    font-weight: bold;
  }

  .ProposalListItemDetail .chart .chartWrap {
    margin-top: 25px;
    position: relative;
  }

  .ProposalListItemDetail #resultChart {
    text-align: center;
  }

  .ProposalListItemDetail .chart .counts {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 400px;
    height: 400px;
    margin-top: -200px;
    margin-left: -200px;
    display: flex;
    align-items: center;
    justify-items: center;
    font-size: 17px;
    font-weight: bold;
  }

  .ProposalListItemDetail .chart .counts > div {
    width: 100%;
  }

  .ProposalListItemDetail .chart .counts .agree {
    color: #1792f0;
    margin-bottom: 5px;
  }

  .ProposalListItemDetail .chart .counts .disagree {
    color: #ed6060;
    margin-top: 5px;
    margin-bottom: 5px;
  }

  .ProposalListItemDetail .chart .counts .abstain {
    color: #728395;
    margin-top: 5px;
  }

  .ProposalListItemDetail .chart .counts .label {
    width: 80px;
    display: inline-block;
    text-align: left;
  }

  .ProposalListItemDetail .chart .counts .votes {
    width: 120px;
    display: inline-block;
    text-align: right;
    color: #333;
  }

  .ProposalListItemDetail .approvalCondition {
    font-size: 12px;
    text-align: left;
  }
</style>
