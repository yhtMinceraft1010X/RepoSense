<template lang="pug">
#summary-charts
  .summary-charts(
    v-for="(repo, i) in filteredRepos",
    v-bind:ref="'summary-charts-' + i",
    v-bind:style="isChartGroupWidgetMode ? {'marginBottom': 0} : {}"
    )
    .summary-charts__title(
      v-if="filterGroupSelection !== 'groupByNone'",
      v-bind:ref="'summary-charts-title-' + i",
      v-bind:class="{ 'active-background': \
        isSelectedGroup(repo[0].name, repo[0].repoName) && !isChartGroupWidgetMode}"
    )
      .summary-charts__title--index(v-if="!isChartGroupWidgetMode") {{ i+1 }}
      .summary-charts__title--groupname(
        v-bind:style="isChartGroupWidgetMode ? {'paddingLeft': 0} : {}"
        )
        template(v-if="filterGroupSelection === 'groupByRepos'") {{ repo[0].repoName }}
        template(
          v-else-if="filterGroupSelection === 'groupByAuthors'",
          v-bind:class=" { warn: repo[0].name === '-' }"
        ) {{ getAuthorDisplayName(repo) }} ({{ repo[0].name }})
      .summary-charts__title--contribution
        .tooltip
          | [{{ getGroupTotalContribution(repo) }} lines]
          span.tooltip-text(
            v-if="filterGroupSelection === 'groupByRepos' && !isChartGroupWidgetMode"
          ) Total contribution of group
          span.tooltip-text(
            v-else-if="filterGroupSelection === 'groupByAuthors' && !isChartGroupWidgetMode"
          ) Total contribution of author
      a(
        v-if="!isGroupMerged(getGroupName(repo)) && !isChartGroupWidgetMode",
        v-on:click="handleMergeGroup(getGroupName(repo))"
      )
        .tooltip
          font-awesome-icon.icon-button(:icon="['fas', 'chevron-up']")
          span.tooltip-text Click to merge group
      a(
        v-if="isGroupMerged(getGroupName(repo)) && !isChartGroupWidgetMode",
        v-on:click="handleExpandGroup(getGroupName(repo))"
      )
        .tooltip
          font-awesome-icon.icon-button(:icon="['fas', 'chevron-down']")
          span.tooltip-text Click to expand group
      a(
        v-if="filterGroupSelection === 'groupByRepos'",
        v-bind:class="!isBrokenLink(getRepoLink(repo[0])) ? '' : 'broken-link'",
        v-bind:href="getRepoLink(repo[0])", target="_blank"
      )
        .tooltip
          font-awesome-icon.icon-button(:icon="getRepoIcon(repo[0])")
          span.tooltip-text(
            v-if="!isChartGroupWidgetMode",
          ) {{getGroupRepoLinkMessage(repo[0])}}
      a(
        v-else-if="filterGroupSelection === 'groupByAuthors'",
        v-bind:class="!isBrokenLink(getAuthorProfileLink(repo[0], repo[0].name)) ? '' : 'broken-link'",
        v-bind:href="getAuthorProfileLink(repo[0], repo[0].name)", target="_blank"
      )
        .tooltip
          font-awesome-icon.icon-button(icon="user")
          span.tooltip-text(
            v-if="!isChartGroupWidgetMode",
          ) {{getAuthorProfileLinkMessage(repo[0])}}
      template(v-if="isGroupMerged(getGroupName(repo))")
        a(
          v-if="filterGroupSelection !== 'groupByAuthors' && !isChartGroupWidgetMode",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabAuthorship(repo[0], repo, 0, isGroupMerged(getGroupName(repo)))"
        )
          .tooltip
            font-awesome-icon.icon-button(
              icon="code",
              v-bind:class="{ 'active-icon': isSelectedTab(repo[0].name, repo[0].repoName, 'authorship', true) }"
            )
            span.tooltip-text Click to view group's code
        a(
          v-if="!isChartGroupWidgetMode",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabZoom(repo[0], filterSinceDate, filterUntilDate, isGroupMerged(getGroupName(repo)))"
        )
          .tooltip
            font-awesome-icon.icon-button(
              icon="list-ul",
              v-bind:class="{ 'active-icon': isSelectedTab(repo[0].name, repo[0].repoName, 'zoom', true) }"
            )
            span.tooltip-text Click to view breakdown of commits
      a(
        v-if="isChartGroupWidgetMode && !isChartWidgetMode",
        v-bind:href="getReportLink()", target="_blank"
      )
        .tooltip
          font-awesome-icon.icon-button(
            icon="arrow-up-right-from-square",
          )
          span.tooltip-text(
            v-if="!isChartGroupWidgetMode",
          ) Click to view breakdown of commits on RepoSense
      a(
        v-if="!isChartGroupWidgetMode",
        v-on:click="getEmbeddedIframe(i)"
      )
        .tooltip(v-bind:id="'tooltip-' + i")
          font-awesome-icon.icon-button(icon="clipboard")
          span.tooltip-text Click to copy iframe link for group

      .tooltip.summary-chart__title--percentile(
          v-if="sortGroupSelection.includes('totalCommits')"
        ) {{ getPercentile(i) }} %&nbsp
        span.tooltip-text.right-aligned {{ getPercentileExplanation(i) }}
    .summary-charts__fileType--breakdown(v-if="filterBreakdown")
      template(v-if="filterGroupSelection !== 'groupByNone'")
        .summary-charts__fileType--breakdown__legend(
          v-for="fileType in getFileTypes(repo)",
          v-bind:key="fileType"
        )
          font-awesome-icon(
            icon="circle",
            v-bind:style="{ 'color': fileTypeColors[fileType] }"
          )
          span &nbsp; {{ fileType }} &nbsp;
    .summary-chart(
      v-for="(user, j) in getRepo(repo)",
      v-bind:style="isChartGroupWidgetMode && j === getRepo(repo).length - 1 ? {'marginBottom': 0} : {}",
      v-bind:ref="'summary-chart-' + j"
      )
      .summary-chart__title(
        v-if="!isGroupMerged(getGroupName(repo))",
        v-bind:class="{ 'active-background': user.name === activeUser && user.repoName === activeRepo \
          && !isChartGroupWidgetMode }"
      )
        .summary-chart__title--index(v-if="!isChartWidgetMode") {{ j+1 }}
        .summary-chart__title--repo(v-if="filterGroupSelection === 'groupByNone'") {{ user.repoName }}
        .summary-chart__title--author-repo(v-if="filterGroupSelection === 'groupByAuthors'") {{ user.repoName }}
        .summary-chart__title--name(
          v-if="filterGroupSelection !== 'groupByAuthors'",
          v-bind:class="{ warn: user.name === '-' }"
        ) {{ user.displayName }} ({{ user.name }})
        .summary-chart__title--contribution.mini-font [{{ user.checkedFileTypeContribution }} lines]
        a(
          v-if="filterGroupSelection !== 'groupByRepos'",
          v-bind:class="!isBrokenLink(getRepoLink(user)) ? '' : 'broken-link'",
          v-bind:href="getRepoLink(user)", target="_blank"
        )
          .tooltip
            font-awesome-icon.icon-button(:icon="getRepoIcon(repo[0])")
            span.tooltip-text(
              v-if="!isChartGroupWidgetMode",
            ) {{getRepoLinkMessage(user)}}
        a(
          v-if="filterGroupSelection !== 'groupByAuthors'",
          v-bind:class="!isBrokenLink(getAuthorProfileLink(user, user.name)) ? '' : 'broken-link'",
          v-bind:href="getAuthorProfileLink(user, user.name)", target="_blank"
        )
          .tooltip
            font-awesome-icon.icon-button(icon="user")
            span.tooltip-text(
              v-if="!isChartGroupWidgetMode",
            ) {{getAuthorProfileLinkMessage(user)}}
        a(
          v-if="!isChartGroupWidgetMode",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabAuthorship(user, repo, j, isGroupMerged(getGroupName(repo)))"
        )
          .tooltip
            font-awesome-icon.icon-button(
              icon="code",
              v-bind:class="{ 'active-icon': isSelectedTab(user.name, user.repoName, 'authorship', false) }"
            )
            span.tooltip-text Click to view author's contribution.
        a(
          v-if="!isChartGroupWidgetMode",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabZoom(user, filterSinceDate, filterUntilDate, isGroupMerged(getGroupName(repo)))"
        )
          .tooltip
            font-awesome-icon.icon-button(
              icon="list-ul",
              v-bind:class="{ 'active-icon': isSelectedTab(user.name, user.repoName, 'zoom', false) }"
            )
            span.tooltip-text Click to view breakdown of commits
        a(
          v-if="isChartGroupWidgetMode",
          v-bind:href="getReportLink()", target="_blank"
        )
          .tooltip
            font-awesome-icon.icon-button(
              icon="arrow-up-right-from-square",
            )
            span.tooltip-text(
              v-if="!isChartGroupWidgetMode",
            ) Click to view breakdown of commits on RepoSense
        a(
          v-if="!isChartGroupWidgetMode",
          v-on:click="getEmbeddedIframe(i , j)"
        )
          .tooltip(v-bind:id="'tooltip-' + i + '-' + j")
            font-awesome-icon.icon-button(icon="clipboard")
            span.tooltip-text Click to copy iframe link
        .tooltip.summary-chart__title--percentile(
          v-if="filterGroupSelection === 'groupByNone' && sortGroupSelection.includes('totalCommits')"
        ) {{ getPercentile(j) }} %&nbsp
          span.tooltip-text.right-aligned {{ getPercentileExplanation(j) }}

      .summary-chart__ramp(
        v-on:click="openTabZoomSubrange(user, $event, isGroupMerged(getGroupName(repo)))"
      )
        c-ramp(
          v-bind:groupby="filterGroupSelection",
          v-bind:user="user",
          v-bind:tframe="filterTimeFrame",
          v-bind:sdate="filterSinceDate",
          v-bind:udate="filterUntilDate",
          v-bind:avgsize="avgCommitSize",
          v-bind:mergegroup="isGroupMerged(getGroupName(repo))",
          v-bind:filtersearch="filterSearch",
          v-bind:is-widget-mode="isChartGroupWidgetMode")
        .overlay

      .summary-chart__contribution
        template(v-if="filterBreakdown")
          .summary-chart__contrib(
            v-for="(widths, fileType) in getFileTypeContributionBars(user.fileTypeContribution)"
          )
            .summary-chart__contrib--bar(
              v-for="width in widths",
              v-bind:style="{ width: `${width}%`,\
                'background-color': fileTypeColors[fileType] }",
              v-bind:title="`${fileType}: ${user.fileTypeContribution[fileType]} lines, \
                total: ${user.checkedFileTypeContribution} lines (contribution from ${minDate} to \
                ${maxDate})`"
            )
        template(v-else)
          .summary-chart__contrib(
            v-bind:title="`Total contribution from ${minDate} to ${maxDate}: \
              ${user.checkedFileTypeContribution} lines`"
          )
            .summary-chart__contrib--bar(
              v-for="width in getContributionBars(user.checkedFileTypeContribution)",
              v-bind:style="{ width: `${width}%` }"
            )
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { mapState } from 'vuex';

import brokenLinkDisabler from '../mixin/brokenLinkMixin';
import cRamp from './c-ramp.vue';
import { Repo, User } from '../types/types';
import { FilterGroupSelection, FilterTimeFrame, SortGroupSelection } from '../types/summary';
import { StoreState, ZoomInfo } from '../types/vuex.d';
import { AuthorFileTypeContributions } from '../types/zod/commits-type';

export default defineComponent({
  name: 'c-summary-charts',
  components: {
    cRamp,
  },
  mixins: [brokenLinkDisabler],
  props: {
    checkedFileTypes: {
      type: Array<string>,
      required: true,
    },
    filtered: {
      type: Array<Array<User>>,
      required: true,
    },
    avgContributionSize: {
      type: Number,
      required: true,
    },
    filterBreakdown: {
      type: Boolean,
      default: false,
    },
    filterGroupSelection: {
      type: String,
      default: FilterGroupSelection.GroupByRepos,
    },
    filterTimeFrame: {
      type: String,
      default: FilterTimeFrame.Commit,
    },
    filterSinceDate: {
      type: String,
      required: true,
    },
    filterUntilDate: {
      type: String,
      required: true,
    },
    isMergeGroup: {
      type: Boolean,
      default: false,
    },
    minDate: {
      type: String,
      required: true,
    },
    maxDate: {
      type: String,
      required: true,
    },
    filterSearch: {
      type: String,
      default: '',
    },
    sortGroupSelection: {
      type: String,
      default: SortGroupSelection.GroupTitle,
    },
    chartGroupIndex: {
      type: Number,
      default: undefined,
    },
    chartIndex: {
      type: Number,
      default: undefined,
    },
  },
  data() {
    return {
      drags: [] as number[],
      activeRepo: null as string | null,
      activeUser: null as string | null,
      activeTabType: null as string | null,
      isTabOnMergedGroup: false,
    };
  },

  computed: {
    avgCommitSize(): number {
      let totalCommits = 0;
      let totalCount = 0;
      this.filteredRepos.forEach((repo) => {
        repo.forEach((user: User) => {
          user.commits?.forEach((slice) => {
            if (slice.insertions > 0) {
              totalCount += 1;
              totalCommits += slice.insertions;
            }
          });
        });
      });
      return totalCommits / totalCount;
    },
    filteredRepos() {
      const repos = this.filtered.filter((repo) => repo.length > 0);
      if (this.isChartGroupWidgetMode && this.chartGroupIndex! < repos.length) {
        return [repos[this.chartGroupIndex!]];
      }
      return repos;
    },
    isChartGroupWidgetMode() {
      return this.chartGroupIndex !== undefined && this.chartGroupIndex >= 0;
    },
    isChartWidgetMode() {
      return this.chartIndex !== undefined && this.chartIndex >= 0 && this.isChartGroupWidgetMode;
    },
    ...mapState({
      mergedGroups: (state: unknown) => (state as StoreState).mergedGroups,
      fileTypeColors: (state: unknown) => (state as StoreState).fileTypeColors,
    }),
  },
  watch: {
    '$store.state.isTabActive': function () {
      if (!this.$store.state.isTabActive) {
        this.removeSelectedTab();
      }
    },
  },
  created() {
    this.retrieveSelectedTabHash();
  },
  methods: {
    getFileTypeContributionBars(fileTypeContribution: AuthorFileTypeContributions): { [key: string]: number[] } {
      let currentBarWidth = 0;
      const fullBarWidth = 100;
      const contributionPerFullBar = (this.avgContributionSize * 2);

      const allFileTypesContributionBars: { [key: string]: number[] } = {};
      if (contributionPerFullBar === 0) {
        return allFileTypesContributionBars;
      }

      Object.keys(fileTypeContribution)
        .filter((fileType) => this.checkedFileTypes.includes(fileType))
        .forEach((fileType) => {
          const contribution = fileTypeContribution[fileType];
          let barWidth = (contribution / contributionPerFullBar) * fullBarWidth;
          const contributionBars = [];

          // if contribution bar for file type is able to fit on the current line
          if (currentBarWidth + barWidth < fullBarWidth) {
            contributionBars.push(barWidth);
            currentBarWidth += barWidth;
          } else {
            // take up all the space left on the current line
            contributionBars.push(fullBarWidth - currentBarWidth);
            barWidth -= fullBarWidth - currentBarWidth;
            // additional bar width will start on a new line
            const numOfFullBars = Math.floor(barWidth / fullBarWidth);
            for (let i = 0; i < numOfFullBars; i += 1) {
              contributionBars.push(fullBarWidth);
            }
            const remainingBarWidth = barWidth % fullBarWidth;
            if (remainingBarWidth > 0) {
              contributionBars.push(remainingBarWidth);
            }
            currentBarWidth = remainingBarWidth;
          }

          allFileTypesContributionBars[fileType] = contributionBars;
        });

      return allFileTypesContributionBars;
    },

    getFileTypes(repo: User[]): string[] {
      const fileTypes: string[] = [];
      repo.forEach((user) => {
        Object.keys(user.fileTypeContribution).forEach((fileType) => {
          if (this.checkedFileTypes.includes(fileType) && !fileTypes.includes(fileType)) {
            fileTypes.push(fileType);
          }
        });
      });
      return fileTypes;
    },

    getGroupTotalContribution(group: User[]): number {
      return group.reduce((acc, ele) => acc + (ele.checkedFileTypeContribution ?? 0), 0);
    },

    getContributionBars(totalContribution: number): number[] {
      const res: number[] = [];
      const contributionLimit = (this.avgContributionSize * 2);
      if (contributionLimit === 0) {
        return res;
      }
      const cnt = Math.floor(totalContribution / contributionLimit);
      for (let cntId = 0; cntId < cnt; cntId += 1) {
        res.push(100);
      }

      const last = (totalContribution % contributionLimit) / contributionLimit;
      if (last !== 0) {
        res.push(last * 100);
      }

      return res;
    },

    getAuthorProfileLink(repo: User, userName: string): string | undefined {
      return window.getAuthorLink(repo.repoId, userName);
    },

    getGroupRepoLinkMessage(repo: User): string {
      return this.getLinkMessage(this.getRepoLink(repo), 'Click to view group\'s repo');
    },

    getAuthorProfileLinkMessage(repo: User): string {
      return this.getLinkMessage(this.getAuthorProfileLink(repo, repo.name), 'Click to view author\'s profile');
    },

    getRepoLinkMessage(repo: User): string {
      return this.getLinkMessage(this.getRepoLink(repo), 'Click to view repo');
    },

    getRepoLink(repo: User): string | undefined {
      const { REPOS } = window;
      const { location, branch } = REPOS[repo.repoId];

      if (Object.prototype.hasOwnProperty.call(location, 'organization')) {
        return window.getBranchLink(repo.repoId, branch);
      }
      this.removeSelectedTab();
      return repo.location;
    },

    getRepoIcon(repo: User): string[] {
      const domainName = window.REPOS[repo.repoId].location.domainName;

      switch (domainName) {
      case 'github':
        return ['fab', 'github'];
      case 'gitlab':
        return ['fab', 'gitlab'];
      case 'bitbucket':
        return ['fab', 'bitbucket'];
      default:
        return ['fas', 'database'];
      }
    },

    // triggering opening of tabs //
    openTabAuthorship(user: User, repo: User[], index: number, isMerged: boolean): void {
      const {
        minDate, maxDate, checkedFileTypes,
      } = this;

      const info = {
        minDate,
        maxDate,
        checkedFileTypes,
        author: user.name,
        repo: user.repoName,
        name: user.displayName,
        isMergeGroup: isMerged,
        location: this.getRepoLink(repo[index]),
        files: [],
      };
      this.addSelectedTab(user.name, user.repoName, 'authorship', isMerged);
      this.$store.commit('updateTabAuthorshipInfo', info);
    },

    openTabZoomSubrange(user: User, evt: MouseEvent, isMerged: boolean): void {
      const isKeyPressed = window.isMacintosh ? evt.metaKey : evt.ctrlKey;

      if (isKeyPressed) {
        if (this.drags.length === 0) {
          this.dragViewDown(evt);
        } else {
          this.dragViewUp(evt);
        }
      }

      // skip if accidentally clicked on ramp chart
      if (this.drags.length === 2 && this.drags[1] - this.drags[0]) {
        const tdiff = (new Date(this.filterUntilDate)).valueOf() - (new Date(this.filterSinceDate)).valueOf();
        const idxs = this.drags.map((x) => (x * tdiff) / 100);
        const tsince = window.getDateStr(new Date(this.filterSinceDate).getTime() + idxs[0]);
        const tuntil = window.getDateStr(new Date(this.filterSinceDate).getTime() + idxs[1]);
        this.drags = [];
        this.openTabZoom(user, tsince, tuntil, isMerged);
      }
    },

    openTabZoom(user: User, since: string, until: string, isMerged: boolean): void {
      const {
        avgCommitSize, filterGroupSelection, filterTimeFrame, filterSearch,
      } = this;
      // Deep copy to ensure changes in zoom (e.g. toggle state) won't affect summary, and vice versa
      const clonedUser = JSON.parse(JSON.stringify(user));
      const info: ZoomInfo = {
        zRepo: user.repoName,
        zAuthor: user.name,
        zFilterGroup: filterGroupSelection,
        zTimeFrame: filterTimeFrame,
        zAvgCommitSize: avgCommitSize,
        zUser: clonedUser,
        zLocation: this.getRepoLink(user),
        zSince: since,
        zUntil: until,
        zIsMerged: isMerged,
        zFileTypeColors: this.fileTypeColors,
        zFromRamp: false,
        zFilterSearch: filterSearch,
      };
      this.addSelectedTab(user.name, user.repoName, 'zoom', isMerged);
      this.$store.commit('updateTabZoomInfo', info);
    },

    async getEmbeddedIframe(chartGroupIndex: number, chartIndex: number = -1) {
      const isChartIndexProvided = chartIndex !== -1;
      // Set height of iframe according to number of charts to avoid scrolling
      let totalChartHeight = 0;
      if (!isChartIndexProvided) {
        totalChartHeight += (this.$refs[`summary-charts-${chartGroupIndex}`] as HTMLElement[])[0].clientHeight;
      } else {
        totalChartHeight += (this.$refs[`summary-chart-${chartIndex}`] as HTMLElement[])[0].clientHeight;
        totalChartHeight += this.filterGroupSelection === 'groupByNone'
          ? 0
          : (this.$refs[`summary-charts-title-${chartGroupIndex}`] as HTMLElement[])[0].clientHeight;
      }

      const margins = 30;
      const iframeStart = '<iframe src="';
      const iframeEnd = `" frameBorder="0" width="800px" height="${totalChartHeight + margins}px"></iframe>`;
      const [baseUrl, ...params] = window.location.href.split('?');
      const groupIndexParam = isChartIndexProvided ? `&chartIndex=${chartIndex}` : '';
      const url = `${baseUrl}#/widget/?${params.join('?')}&chartGroupIndex=${chartGroupIndex}${groupIndexParam}`;
      const iframe = iframeStart + url + iframeEnd;
      if (navigator.clipboard) {
        navigator.clipboard.writeText(iframe);
      } else {
        // Clipboard API is not supported (non-secure origin of neither HTTPS nor localhost)
        const textarea = document.createElement('textarea');
        textarea.value = iframe;
        textarea.setAttribute('readonly', '');
        textarea.style.position = 'absolute';
        textarea.style.left = '-9999px';
        document.body.appendChild(textarea);
        textarea.select();
        document.execCommand('copy');
        document.body.removeChild(textarea);
      }
      const tooltipId = `tooltip-${chartGroupIndex}${isChartIndexProvided ? `-${chartIndex}` : ''}`;
      this.updateCopyTooltip(tooltipId, 'Copied iframe');
    },
    updateCopyTooltip(tooltipId: string, text: string) {
      const tooltipElement = document.getElementById(tooltipId);
      if (tooltipElement && tooltipElement.querySelector('.tooltip-text')) {
        const tooltipTextElement = tooltipElement.querySelector('.tooltip-text');
        const originalText = tooltipTextElement!.textContent;
        tooltipElement.querySelector('.tooltip-text')!.textContent = text;
        setTimeout(() => {
          tooltipTextElement!.textContent = originalText;
        }, 2000);
      }
    },
    getReportLink() {
      const url = window.location.href;
      const regexToRemoveWidget = /([?&])((chartIndex|chartGroupIndex)=\d+)/g;
      return url.replace(regexToRemoveWidget, '');
    },
    getRepo(repo: Repo[]) {
      if (this.isChartGroupWidgetMode && this.isChartWidgetMode) {
        return [repo[this.chartIndex!]];
      }
      return repo;
    },

    getBaseTarget(target: HTMLElement | null): HTMLElement | null {
      if (!target) {
        // Should never reach here - function assumes that target is a child of the div with class 'summary-chart__ramp'
        console.error('Error: The getBaseTarget function in c-summary-charts.vue has been called on an element that is '
        + 'not a child of the div with class summary-chart__ramp. This might affect the drag view functionality.');
        return null;
      }
      return target.className === 'summary-chart__ramp'
          ? target
          : this.getBaseTarget(target.parentElement);
    },

    dragViewDown(evt: MouseEvent): void {
      window.deactivateAllOverlays();

      const pos = evt.clientX;
      const ramp = this.getBaseTarget(evt.target as HTMLElement);
      if (!ramp || !ramp.parentElement) {
        return;
      }
      this.drags = [pos];

      const base = ramp.offsetWidth;
      const offset = ramp.parentElement.offsetLeft;

      const overlay = ramp.getElementsByClassName('overlay')[0] as HTMLElement;
      overlay.style.marginLeft = '0';
      overlay.style.width = `${((pos - offset) * 100) / base}%`;
      overlay.className += ' edge';
    },

    dragViewUp(evt: MouseEvent): void {
      window.deactivateAllOverlays();
      const ramp = this.getBaseTarget(evt.target as HTMLElement);
      if (!ramp || !ramp.parentElement) {
        return;
      }

      const base = ramp.offsetWidth;
      this.drags.push(evt.clientX);
      this.drags.sort((a, b) => a - b);

      const offset = ramp.parentElement.offsetLeft;
      this.drags = this.drags.map((x) => ((x - offset) * 100) / base);

      const overlay = ramp.getElementsByClassName('overlay')[0] as HTMLElement;
      overlay.style.marginLeft = `${this.drags[0]}%`;
      overlay.style.width = `${this.drags[1] - this.drags[0]}%`;
      overlay.className += ' show';
    },

    getPercentile(index: number): string {
      if (this.filterGroupSelection === FilterGroupSelection.GroupByNone) {
        return (Math.round(((index + 1) * 1000) / this.filtered[0].length) / 10).toFixed(1);
      }
      return (Math.round(((index + 1) * 1000) / this.filtered.length) / 10).toFixed(1);
    },

    getGroupName(group: User[]): string {
      return window.getGroupName(group, this.filterGroupSelection);
    },

    isGroupMerged(groupName: string): boolean {
      return this.mergedGroups.includes(groupName);
    },

    handleMergeGroup(groupName: string): void {
      const info = this.mergedGroups;
      info.push(groupName);
      this.$store.commit('updateMergedGroup', info);
    },

    handleExpandGroup(groupName: string): void {
      const info = this.mergedGroups.filter((x) => x !== groupName);
      this.$store.commit('updateMergedGroup', info);
    },

    getAuthorDisplayName(repo: User[]): string {
      return window.getAuthorDisplayName(repo);
    },

    retrieveSelectedTabHash(): void {
      const hash = window.hashParams;

      if (hash.tabAuthor) {
        this.activeUser = hash.tabAuthor;
      } else if (hash.zA) {
        this.activeUser = hash.zA;
      }

      if (hash.tabRepo) {
        this.activeRepo = hash.tabRepo;
      } else if (hash.zR) {
        this.activeRepo = hash.zR;
      }

      if (hash.isTabOnMergedGroup) {
        if (this.filterGroupSelection === FilterGroupSelection.GroupByAuthors) {
          this.activeRepo = null;
        } else if (this.filterGroupSelection === FilterGroupSelection.GroupByRepos) {
          this.activeUser = null;
        }
        this.isTabOnMergedGroup = true;
      }

      if (hash.tabType) {
        this.activeTabType = hash.tabType;
      }
    },

    addSelectedTab(userName: string, repo: string, tabType: string, isMerged: boolean): void {
      if (!isMerged || this.filterGroupSelection === FilterGroupSelection.GroupByAuthors) {
        this.activeUser = userName;
      } else {
        this.activeUser = null;
      }

      if (isMerged && this.filterGroupSelection === FilterGroupSelection.GroupByAuthors) {
        this.activeRepo = null;
      } else {
        this.activeRepo = repo;
      }

      if (isMerged) {
        window.addHash('isTabOnMergedGroup', 'true');
        this.isTabOnMergedGroup = true;
      } else {
        window.removeHash('isTabOnMergedGroup');
        this.isTabOnMergedGroup = false;
      }

      this.activeTabType = tabType;
      window.encodeHash();
    },

    removeSelectedTab(): void {
      this.activeUser = null;
      this.activeRepo = null;
      this.activeTabType = null;

      window.removeHash('isTabOnMergedGroup');
      window.encodeHash();
    },

    isSelectedTab(userName: string, repo: string, tabType: string, isMerged: boolean): boolean {
      if (!isMerged) {
        return this.activeUser === userName && this.activeRepo === repo
            && this.activeTabType === tabType;
      }

      if (this.filterGroupSelection === FilterGroupSelection.GroupByAuthors) {
        return this.isTabOnMergedGroup && this.activeUser === userName
            && this.activeTabType === tabType;
      }

      return this.isTabOnMergedGroup && this.activeRepo === repo
          && this.activeTabType === tabType;
    },

    isSelectedGroup(userName: string, repo: string): boolean {
      return this.isTabOnMergedGroup
          && ((this.filterGroupSelection === FilterGroupSelection.GroupByRepos && this.activeRepo === repo)
          || (this.filterGroupSelection === FilterGroupSelection.GroupByAuthors && this.activeUser === userName));
    },

    getPercentileExplanation(j: number): string {
      const explanation = `Based on the current sorting order, this item is in the top ${this.getPercentile(j)}%`;
      return explanation;
    },

  },
});
</script>
