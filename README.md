<div class="admin-container">
  <div class="card">
    <h2>General Properties</h2>
    <div class="form-grid">
      <div class="form-group">
        <label>Max Logon Attempts</label>
        <input formControlName="maxLogonAttempts" type="text" />
      </div>

      <div class="form-group">
        <label>Select SCD Domain Name</label>
        <select formControlName="domainName">
          <option value="EURO_DOMAIN">EURO_DOMAIN</option>
          <option value="SCD_DOMAIN">SCD_DOMAIN</option>
        </select>
      </div>

      <div class="form-group">
        <label>Report Service General Folder</label>
        <input formControlName="reportFolder" type="text" />
      </div>

      <div class="form-group">
        <label>Purge Retention Count (for Logins)</label>
        <input formControlName="purgeRetentionDays" type="text" />
      </div>
    </div>
  </div>

  <div class="card">
    <h2>Service Application Type Properties</h2>
    <div class="form-grid">
      <div class="form-group">
        <label>Max Record Count</label>
        <input formControlName="maxRecordCount" type="text" />
      </div>

      <div class="form-group">
        <label>Max Execution Range</label>
        <input formControlName="maxExecutionRange" type="text" />
      </div>

      <div class="form-group">
        <label>Rows Per Page</label>
        <input formControlName="rowsPerPage" type="text" />
      </div>

      <div class="form-group">
        <label>Reserve Per Query</label>
        <input formControlName="reservePerQuery" type="text" />
      </div>

      <div class="form-group">
        <label>Items to Select</label>
        <input formControlName="itemsToSelect" type="text" />
      </div>

      <div class="form-group">
        <label>Selector Threshold</label>
        <input formControlName="selectorThreshold" type="text" />
      </div>
    </div>
  </div>

  <div class="card">
    <h2>Additional Properties</h2>
    <div class="form-grid">
      <div class="form-group">
        <label>Select SCD Domain Name</label>
        <select formControlName="domainName2">
          <option value="EURO_DOMAIN">EURO_DOMAIN</option>
          <option value="SCD_DOMAIN">SCD_DOMAIN</option>
        </select>
      </div>

      <div class="form-group">
        <label>Report Folder</label>
        <input formControlName="reportFolder2" type="text" />
      </div>

      <div class="form-group">
        <label>Purge Retention Days</label>
        <input formControlName="purgeRetentionDays2" type="text" />
      </div>
    </div>
  </div>
</div>



.admin-container {
  padding: 20px;
  max-width: 1200px;
  margin: auto;
  font-family: Arial, sans-serif;
}

.card {
  border: 1px solid #ccc;
  border-radius: 10px;
  padding: 20px;
  margin-bottom: 30px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

h2 {
  margin-bottom: 20px;
  font-size: 20px;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
}

label {
  font-weight: bold;
  margin-bottom: 5px;
}

input, select {
  padding: 8px;
  border: 1px solid #bbb;
  border-radius: 5px;
}import { ReactiveFormsModule } from '@angular/forms';

@NgModule({
  imports: [
    ReactiveFormsModule,
    // ...
  ],
})
export class AppModule {}
