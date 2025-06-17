sidebar.component.ts
ts
Copy code
import { Component, Input, OnChanges } from '@angular/core';

@Component({
  selector: 'app-sidebar',
  templateUrl: './sidebar.component.html',
  styleUrls: ['./sidebar.component.css']
})
export class SidebarComponent implements OnChanges {
  @Input() isCollapsed: boolean = false;

  selectedMenu: string | null = null;

  toggleSubmenu(menu: string) {
    this.selectedMenu = menu;
  }

  ngOnChanges() {
    if (this.isCollapsed) {
      this.selectedMenu = null; // auto-close submenu when main sidebar collapses
    }
  }
}
✅ sidebar.component.html
html
Copy code
<!-- White Main Sidebar -->
<div class="main-sidebar" [class.collapsed]="isCollapsed">
  <ul>
    <li (click)="toggleSubmenu('reference')">Reference Data</li>
    <li (click)="toggleSubmenu('currency')">Currency</li>
  </ul>
</div>

<!-- Green Submenu Sidebar -->
<div class="submenu-sidebar" *ngIf="selectedMenu">
  <ng-container [ngSwitch]="selectedMenu">
    <div *ngSwitchCase="'reference'">Reference Panel Content</div>
    <div *ngSwitchCase="'currency'">Currency Panel Content</div>
  </ng-container>
</div>
✅ sidebar.component.css
css
Copy code
.main-sidebar {
  width: 250px;
  background-color: white;
  height: 100vh;
  transition: width 0.3s;
  float: left;
  border-right: 1px solid #ccc;
  padding: 1rem;
}

.main-sidebar.collapsed {
  width: 80px;
}

.main-sidebar ul {
  list-style: none;
  padding: 0;
}

.main-sidebar li {
  cursor: pointer;
  padding: 0.5rem 0;
  font-weight: bold;
}

.submenu-sidebar {
  position: absolute;
  left: 250px;
  width: 250px;
  height: 100vh;
  background-color: lightgreen;
  padding: 1rem;
  transition: all 0.3s;
}
✅ Usage in layout.component.html
html
Copy code
<app-header (sidebarToggle)="onSidebarToggle()"></app-header>

<div class="main-area">
  <app-sidebar [isCollapsed]="isCollapsed"></app-sidebar>
  <div class="content-area">
    <router-outlet></router-outlet>
  </div>
</div>
✅ layout.component.ts
ts
Copy code
export class LayoutComponent {
  isCollapsed: boolean = false;

  onSidebarToggle() {
    this.isCollapsed = !this.isCollapsed;
  }
}
Let me know if you want a working StackBlitz demo or want submenu to slide in/out instead of showing/hiding.







