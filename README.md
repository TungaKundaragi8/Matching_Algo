Perfect! You want the left AG Grid table (for Accounting Period Groups) to have:

1. ✅ A checkbox for row selection


2. ✅ A custom header with:

➤ button (next arrow)

Copy button

Audit button




All of these should appear inside one column header (not separate ones), and cell-wise ➤ in each row.


---

✅ Final Updated Solution

🔧 1. columnDefs for Left AG Grid (with Checkbox + Custom Header + ➤ Cell)

columnDefs = [
  {
    headerName: '',
    field: 'actions',
    width: 120,
    headerComponentFramework: CustomHeaderComponent,
    cellRenderer: () => '➤'
  },
  {
    headerName: 'Name',
    field: 'name',
    editable: true,
    checkboxSelection: true,
    headerCheckboxSelection: true
  }
];


---

🎯 2. Custom Header Component (CustomHeaderComponent)

📁 custom-header.component.ts

import { Component } from '@angular/core';

@Component({
  selector: 'app-custom-header',
  template: `
    <div class="header-actions">
      <button (click)="onAudit()" title="Audit">📝</button>
      <button (click)="onCopy()" title="Copy">📋</button>
      <span class="arrow">➤</span>
    </div>
  `,
  styles: [`
    .header-actions {
      display: flex;
      justify-content: space-around;
      align-items: center;
    }
    .header-actions button {
      background: none;
      border: none;
      cursor: pointer;
      font-size: 14px;
    }
    .arrow {
      font-size: 18px;
    }
  `]
})
export class CustomHeaderComponent {
  onAudit() {
    alert('Audit clicked');
  }

  onCopy() {
    alert('Copy clicked');
  }
}


---

📁 Update app.module.ts to declare the component

import { CustomHeaderComponent } from './custom-header.component';

@NgModule({
  declarations: [
    AppComponent,
    CustomHeaderComponent // 👈 declare it here
  ],
  imports: [
    BrowserModule,
    AngularSplitModule,
    AgGridModule.withComponents([CustomHeaderComponent]) // 👈 register
  ]
})
export class AppModule { }


---

✅ 3. Sample Left Table Output

✅	Name

➤	Q1 2025
➤	Q2 2025


And in the header above ➤ you’ll see:

📝 | 📋 | ➤
(Audit | Copy | Arrow)


---

🧪 Bonus: Add rowSelection="multiple" in AG Grid

<ag-grid-angular
  ...
  rowSelection="multiple"
  ...
>


---

Let me know if you want the Copy button to copy selected rows to clipboard, or Audit to open a modal.
