<template>
  <v-tooltip top>
    <template #activator="{on}">
      <IconLinkItem
        key="exportCaat"
        text="CAAT Spreadsheet"
        icon="mdi-file-excel"
        @click="exportCaat()"
        v-on="on"
      />
    </template>
    <span>Compliance Assessment Audit Tracking Data</span>
  </v-tooltip>
</template>

<script lang="ts">
import IconLinkItem from '@/components/global/sidebaritems/IconLinkItem.vue';
import {s2ab} from '@/utilities/export_util';
import {saveAs} from 'file-saver';
import {ContextualizedControl, HDFControl, HDFControlSegment} from 'inspecjs';
import _ from 'lodash';
import Vue from 'vue';
import Component from 'vue-class-component';
import {Prop} from 'vue-property-decorator';
import XLSX from 'xlsx';
import {Filter, FilteredDataModule} from '../../store/data_filters';
import {InspecDataModule} from '../../store/data_store';

type CAATRow = {
  [key: string]: string | undefined;
};

@Component({
  components: {
    IconLinkItem
  }
})
export default class ExportCaat extends Vue {
  @Prop({type: Object, required: true}) readonly filter!: Filter;

  MAX_CELL_SIZE = 32000;

  header = [
    'Control Number',
    'Finding Title',
    'Date Identified',
    'Finding ID',
    'Information System or Program Name',
    'Repeat Findings',
    'Repeat Finding Weakness ID',
    'Finding Description',
    'Weakness Description',
    'Control Weakness Type',
    'Source',
    'Assessment/Audit Company',
    'Test Method',
    'Test Objective',
    'Test Result Description',
    'Test Result',
    'Recommended Corrective Action(s)',
    'Effect on Business',
    'Likelihood',
    'Impact'
  ];

  takenFilenames: string[] = [];

  fileSettings = {
    Title: 'Compliance Assessment/Audit Tracking (CAAT) Spreadsheet',
    Subject: 'Assessment Data',
    Author: 'Heimdall',
    CreatedDate: new Date()
  };

  getRow(
    control: ContextualizedControl,
    filename: string | undefined
  ): CAATRow[] {
    const hdf = control.hdf;
    const allRows: CAATRow[] = [];
    for (const formatted of hdf.canonized_nist({
      max_specifiers: 3,
      pad_zeros: true,
      allow_letters: false,
      add_spaces: false
    })) {
      // Establish our row
      if (!formatted) {
        continue;
      }
      const row: CAATRow = {};
      row['Control Number'] = formatted;
      row['Finding Title'] = `Test ${this.fix(hdf.wraps.id)} - ${this.fix(
        hdf.wraps.title
      )}`;
      if (hdf.start_time) {
        row['Date Identified'] = this.convertDate(
          new Date(hdf.start_time),
          '/'
        );
      }
      row['Finding ID'] = `${filename} - Test ${this.fix(hdf.wraps.id)}`;
      row['Finding Description'] = this.fix(hdf.wraps.title);
      row['Weakness Description'] = this.createCaveat(hdf);
      row['Control Weakness Type'] = 'Security';
      row['Source'] = 'Self-Assessment';
      row['Test Method'] = 'Test';
      row['Test Objective'] = this.fix(
        hdf.descriptions.check || hdf.wraps.tags.check
      );
      row['Test Result Description'] = this.fix(
        this.createTestResultDescription(hdf)
      );
      row['Test Result'] = this.createTestResult(hdf);
      row['Recommended Corrective Action(s)'] = this.fix(
        hdf.descriptions.fix || hdf.wraps.tags.fix
      );
      row['Impact'] = this.createImpact(hdf);
      allRows.push(row);
    }
    return allRows;
  }

  exportCaat() {
    // Define our workbook
    const wb = XLSX.utils.book_new();
    // For each file in our filter
    this.filter.fromFile.forEach((fileId) => {
      // Find our file within InspecDataModule
      const file = InspecDataModule.allFiles.find((f) => f.uniqueId === fileId);
      let renameCount = 2;
      let sheetName: string = `${file?.filename || fileId}`.substring(0, 31);
      while (this.takenFilenames.includes(sheetName)) {
        sheetName =
          `${file?.filename || fileId} `.substring(0, 26) + ` (${renameCount})`;
        renameCount++;
      }
      this.takenFilenames.push(sheetName);
      // Create a new Sheet
      wb.SheetNames.push(sheetName);
      wb.Props = this.fileSettings;
      // Get the controls for the current file
      const controls = FilteredDataModule.controls({
        ...this.filter,
        fromFile: [fileId]
      });
      const hitIds = new Set();
      const rows: CAATRow[] = [];
      // Convert them into rows
      for (const ctrl of controls) {
        const root = ctrl.root;
        if (hitIds.has(root.hdf.wraps.id)) {
          continue;
        } else {
          hitIds.add(root.hdf.wraps.id);
          rows.push(...this.getRow(root, file?.filename));
        }
      }
      // Add rows to sheet
      const ws = XLSX.utils.json_to_sheet(rows, {header: this.header});
      wb.Sheets[sheetName] = ws;
    });

    const wbout = XLSX.write(wb, {bookType: 'xlsx', type: 'binary'});
    saveAs(
      new Blob([s2ab(wbout)], {type: 'application/octet-stream'}),
      `CAAT-${this.convertDate(new Date(), '-')}.xlsx`
    );
  }

  createCaveat(hdf: HDFControl): string {
    const caveat = hdf.descriptions.caveat
      ? `(Caveat: ${this.fix(hdf.descriptions.caveat)})\n`
      : '';
    return caveat + this.fix(hdf.wraps.desc);
  }

  // Create Test Result Description
  createTestResult(hdf: HDFControl): string {
    return hdf.status === 'Passed' ? 'Satisfied' : 'Other Than Satisfied';
  }

  // Create Test Result Description
  createTestResultDescription(hdf: HDFControl): string {
    let testResult = `${hdf.status}:\r\n\r\n`;
    _.get(hdf, 'wraps.results').forEach((result: HDFControlSegment) => {
      if (result.message) {
        testResult += `${result.status.toUpperCase()} -- Test: ${
          result.code_desc
        }\r\nMessage: ${result.message}\r\n\r\n`;
      } else {
        testResult += `${result.status.toUpperCase()} -- Test: ${
          result.code_desc
        }\r\n\r\n`;
      }
    });
    return testResult;
  }

  createImpact(hdf: HDFControl): string {
    const controlSeverity =
      hdf.severity === 'medium' ? 'moderate' : hdf.severity;
    return this.fix(hdf.wraps.impact === 0 ? 'none' : controlSeverity);
  }

  fix(x: string | null | undefined) {
    return (x || '')
      .replace(/(\r\n|\n|\r)/gm, '\r\n')
      .slice(0, this.MAX_CELL_SIZE);
  }

  /** Outputs the given number as a 2-digit string. Brittle **/
  padTwoDigits(s: number): string {
    return s < 10 ? `0${s}` : `${s}`;
  }

  convertDate(d: Date, delimiter: string): string {
    return [
      this.padTwoDigits(d.getMonth() + 1),
      this.padTwoDigits(d.getDate()),
      d.getFullYear()
    ].join(delimiter);
  }
}
</script>
