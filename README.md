# AdWords_MCC_Link-Checker
Check URL health of top performing keywords
/**
* The URL of the tracking spreadsheet.
* This should be a copy of https://goo.gl/gr2UaG
*/
var SPREADSHEET_URL = 'https://docs.google.com/spreadsheet/ccc?key=0AraoMyUpWsDydG15bmF3LTduWFNHd3RLWUZERURFMWc';

function main() {
  var spreadsheet = SpreadsheetApp.openByUrl(SPREADSHEET_URL);
  var settings = readSettings(spreadsheet);
  var lastRun = getLastRunDate(spreadsheet);
  var today = new Date();

  var resumingPreviousRun =
      (today.getYear() == lastRun.getYear() &&
      today.getMonth() == lastRun.getMonth() &&
      today.getDate() == lastRun.getDate());
  
  var accountIterator = getAccountSelector(settings).get();

  while (accountIterator.hasNext()) {
    var account = accountIterator.next();
    ensureSheetExistsForAccount(account, spreadsheet);
    
    if (!resumingPreviousRun) {
      Logger.log('Fresh run...');
//      sendEmails(settings, 1000000);
      removeLinkCheckerLabels(account, settings);
      clearAccountSheet(account, spreadsheet);
      createLinkCheckerLabels(account, settings);
    }
  }

  saveLastRunDate(today, spreadsheet);
  accountSelector = getAccountSelector(settings);
  accountSelector.executeInParallel('processAccountInParallel',
      'afterProcessCallback');
}

/**
* Read the link checker script settings from the dashboard.
*
* @param {Object} spreadsheet The link checker spreadsheet object.
* @return {Object} A dictionary with setting name and setting value as key
*     value pairs.
*/
function readSettings(spreadsheet) {
  var sheetName = 'Dashboard';
  var dashboardSheet = spreadsheet.getSheetByName(sheetName);
  var settings = {
    /* Settings to control what entities should be checked. */
    'checkKeywords': dashboardSheet.getRange(4, 3).getValue() == 'Yes',
    'checkAds': dashboardSheet.getRange(5, 3).getValue() == 'Yes',
    'checkSitelinks': dashboardSheet.getRange(6, 3).getValue() == 'Yes',

    /* Setting to control the label name to be applied for checked entities. */
    'keywordsLabel': dashboardSheet.getRange(7, 3).getValue(),
    'adsLabel': dashboardSheet.getRange(8, 3).getValue(),
    'sitelinksLabel': dashboardSheet.getRange(9, 3).getValue(),

    /* Email settings. */
    'emailToAddresses': dashboardSheet.getRange(12, 3).getValue(),
    'emailCCAddresses': dashboardSheet.getRange(13, 3).getValue(),

    /* Accounts should be checked. */
    'accountsToCheck': dashboardSheet.getRange(16, 3).getValue()
  };
  var accountsToCheck = settings.accountsToCheck.split(',');
  settings.accountsToCheck = [];

  for (var i = 0; i < accountsToCheck.length; i++) {
    var accountToCheck = accountsToCheck[i].trim();
    if (accountToCheck.length > 0) {
      settings.accountsToCheck.push(accountToCheck);
    }
  }
  return settings;
}

/**
* Get the last date on which this script was run.
*
* @param {Object} spreadsheet The link checker spreadsheet object.
* @return {Date} The date on which this script ran last, or today's date
*     if the date isn't available in the spreadsheet.
*/
function getLastRunDate(spreadsheet) {
  var summarySheet = getSummarySheet(spreadsheet);
  var lastRun = summarySheet.getRange(1, 2).getValue();
  if (!lastRun) {
    lastRun = new Date(0, 0, 0);
  }
  return lastRun;
}

/**
* Gets an AccountSelector object for enumerating accounts.
* @param {Object} settings The settings read from the link checker spreadsheet.
* @return {Object} An AccountSelector object built using the ids of accounts
*     from the settings. If no accounts were specified in settings, then an
*     empty selector is returned, that enumerates all child accounts under
*     the MCC account.
*/
function getAccountSelector(settings) {
  var accountSelector = MccApp.accounts();
  if (settings.accountsToCheck.length > 0) {
    accountSelector = accountSelector.withIds(settings.accountsToCheck);
  }
  return accountSelector;
}

/**
* Ensure that a tracking sheet exists on the link checker spreadsheet for an
* account being checked.
*
* @param {Object} account The account for which a tracking sheet should exist.
* @param {Object} spreadsheet The link checker spreadsheet object.
*/
function ensureSheetExistsForAccount(account, spreadsheet) {
  var accountSheet = getSheetForAccount(account, spreadsheet);
  if (accountSheet == null) {
    var templateSheet = spreadsheet.getSheetByName('Report template');
    accountSheet = templateSheet.copyTo(spreadsheet);
    accountSheet.setName(account.getCustomerId());
    spreadsheet.setActiveSheet(accountSheet);
    spreadsheet.moveActiveSheet(spreadsheet.getSheets().length);
  }
  accountSheet.getRange(2, 5).setValue(account.getCustomerId());
}

/**
* Remove labels applied by link checker script from a previous run.
*
* @param {Object} account The account from which the labels should be removed.
* @param {Object} settings The settings read from the link checker spreadsheet.
*/
function removeLinkCheckerLabels(account, settings) {
  var mccAccount = AdWordsApp.currentAccount();
  MccApp.select(account);

  var labelNames = [settings.keywordsLabel, settings.adsLabel,
      settings.sitelinksLabel];

  for (var i = 0; i < labelNames.length; i++) {
    var labelIterator = AdWordsApp.labels().withCondition("LabelName = '" +
        labelNames[i] + "'").get();
    Logger.log('Found matching labels: %s', labelIterator.totalNumEntities());
    while (labelIterator.hasNext()) {
      Logger.log('Remove label...');
      var label = labelIterator.next();
      label.remove();
    }
  }
  MccApp.select(mccAccount);
}

/**
* Create labels for a fresh run of link checker script.
*
* @param {Object} account The account to which the labels should be added.
* @param {Object} settings The settings read from the link checker spreadsheet.
*/
function createLinkCheckerLabels(account, settings) {
  var mccAccount = AdWordsApp.currentAccount();
  MccApp.select(account);
  var labelNames = [settings.keywordsLabel, settings.adsLabel,
      settings.sitelinksLabel];

  for (var i = 0; i < labelNames.length; i++) {
    Logger.log('Create label %s in %s', labelNames[i], account.getCustomerId());

    // If you need labels with a different color, you can customize them here.
    AdWordsApp.createLabel(labelNames[i], 'Created by link checker', '#ff0000');
    // Force AdWords to flush cache.
    var label = AdWordsApp.labels().withCondition("LabelName = '" +
        labelNames[i] + "'").get().next();
  }
  MccApp.select(mccAccount);
}

/**
* Clears the tracking sheet for an account in the link checker spreadsheet.
* @param {Object} account The account for which tracking sheet should be
*     cleared.
* @param {Object} spreadsheet The link checker spreadsheet object.
*/
function clearAccountSheet(account, spreadsheet) {
  var accountSheet = getSheetForAccount(account, spreadsheet);
  // Delete all the rows > 4.
  if (accountSheet.getMaxRows() > 4) {
    Logger.log('Deleting %s rows from 4', accountSheet.getMaxRows() - 4);
    accountSheet.deleteRows(4, accountSheet.getMaxRows() - 4);
  }
  // Clear row 4.
  if (accountSheet.getMaxRows() > 3) {
    accountSheet.getRange(4, 1, 1, 8).setValue('');
  }
}

/**
* Gets the summary sheet.
*
* @param {Object} spreadsheet The link checker spreadsheet object.
* @return {Object} The summary sheet object from the link checker spreadsheet.
*/
function getSummarySheet(spreadsheet) {
  return spreadsheet.getSheetByName('Dashboard');
}

/**
* Save the last run date of the link checker script on the summary sheet of the
* link checker spreadsheet.
*
* @param {Date} lastRun The last run date.
* @param {Object} spreadsheet The link checker spreadsheet object.
*/
function saveLastRunDate(lastRun, spreadsheet) {
  var summarySheet = getSummarySheet(spreadsheet);
  summarySheet.getRange(1, 2).setValue(lastRun);
}

/**
* The entry point for the link checker script when processing accounts in
* parallel. This method is called by the executeInParallel method for each
* account in the account selector, and in parallel.
*/
function processAccountInParallel() {
  var spreadsheet = SpreadsheetApp.openByUrl(SPREADSHEET_URL);
  var settings = readSettings(spreadsheet);
  var account = AdWordsApp.currentAccount();
  var accountSheet = getSheetForAccount(account, spreadsheet);
  checkDestinationUrls(accountSheet, settings);
}

/**
* Gets the tracking sheet in the link checker spreadsheet for an account.
*
* @param {Object} account The account for which links are being checked.
* @param {Object} spreadsheet The link checker spreadsheet object.
* @return {Object} The tracking sheet object for the account.
*/
function getSheetForAccount(account, spreadsheet) {
  return spreadsheet.getSheetByName(account.getCustomerId().toString());
}

/**
* Checks the destination urls of the currently selected account.
*
* @param {Object} accountSheet The tracking sheet for the current account in
*     the link checker spreadsheet.
* @param {Object} settings The settings read from the link checker spreadsheet.
*/
function checkDestinationUrls(accountSheet, settings) {
  if (settings.checkKeywords) {
    var keywords = AdWordsApp.keywords()
        .forDateRange("LAST_14_DAYS")
        .withCondition("Status = 'ENABLED'")
        .withCondition("DestinationUrl != ''")
        .withCondition("AdGroupStatus = 'ENABLED'")
        .withCondition("CampaignStatus = 'ENABLED'")
        .withCondition("LabelNames CONTAINS_NONE ['" + settings.keywordsLabel + "']")
        .orderBy('Clicks DESC')
        .withLimit(300)
        .get();
    Logger.log('Checking %s keywords.', keywords.totalNumEntities());
    Logger.log('Checking this account: %s', AdWordsApp.currentAccount().getName());
    while (keywords.hasNext()) {
      var keyword = keywords.next();
      var finalUrl = keyword.urls().getFinalUrl();
      var status = getUrlStatus(finalUrl);
      accountSheet.appendRow(['', finalUrl,
                              status, keyword.getCampaign().getName(),
                              keyword.getAdGroup().getName(),
                              keyword.getText(), '', '']);
      keyword.applyLabel(settings.keywordsLabel);
    }
  }

  if (settings.checkAds) {
    var ads = AdWordsApp.ads()
        .forDateRange("LAST_14_DAYS")
        .withCondition("Status = 'ENABLED'")
        .withCondition("DestinationUrl != ''")
        .withCondition("AdGroupStatus = 'ENABLED'")
        .withCondition("CampaignStatus = 'ENABLED'")
        .withCondition("LabelNames CONTAINS_NONE['" + settings.adsLabel + "']")
        .orderBy('Clicks DESC')
        .get();
    Logger.log('Checking %s ads.', ads.totalNumEntities());
    while (ads.hasNext()) {
      var ad = ads.next();
      var finalUrl = ad.urls().getFinalUrl();
      var status = getUrlStatus(finalUrl);
      accountSheet.appendRow(['', ad.getDestinationUrl(),
                              status, ad.getCampaign().getName(),
                              ad.getAdGroup().getName(),
                              '', ad.getHeadline(), '']);
      ad.applyLabel(settings.adsLabel);
    }
  }

  if (settings.checkSitelinks) {
    var campaigns = AdWordsApp.campaigns()
        .withCondition("LabelNames CONTAINS_NONE['" + settings.sitelinksLabel +
            "']")
        .get();
    while (campaigns.hasNext()) {
      var campaign = campaigns.next();
      var sitelinks = campaign.extensions().sitelinks().get();

      Logger.log('Checking %s sitelinks.', sitelinks.totalNumEntities());
      while (sitelinks.hasNext()) {
        var sitelink = sitelinks.next();
      var finalUrl = sitelink.urls().getFinalUrl();
      var status = getUrlStatus(finalUrl);
        accountSheet.appendRow(['', sitelink.getLinkUrl(),
                                status, campaign.getName(),
                                '', '', '', sitelink.getLinkText()]);
      }
      campaign.applyLabel(settings.sitelinksLabel);
    }
 }
}

/**
* Gets the status of an url.
*
* @param {string} url The url to check.
* @return {Integer} The status code for the url.
*/
function getUrlStatus(url) {
  url = encodeURI(url);
  var response = UrlFetchApp.fetch(url, { muteHttpExceptions: true});
  return response.getResponseCode();
}

/**
* The callback method for link checker script after processing all the
* accounts. This method is called by the executeInParallel method after all the
* parallel methods return either due to completion, or timeouts.
*/
function afterProcessCallback() {
  var spreadsheet = SpreadsheetApp.openByUrl(SPREADSHEET_URL);
  var settings = readSettings(spreadsheet);

  clearSummarySheet(spreadsheet);
  var badUrlCount = updateCheckedUrlSummary(spreadsheet);
  sendEmails(settings, badUrlCount);
}

/**
* Send emails about completed link checks.
*
* @param {Object} settings The settings read from the link checker spreadsheet. 
*/
function sendEmails(settings, errorUrls) {
  if (errorUrls != 0) {
  if (settings.emailToAddresses != '') {
    var options = {};
    if (settings.emailCCAddresses != '') {
      options['cc'] = settings.emailCCAddresses;
    }
    MailApp.sendEmail(settings.emailToAddresses, 'MCC Link Checker Script',
                      'MCC Link Checker Script ran successfully and found ' +
                      errorUrls + ' errors. See ' +
                      SPREADSHEET_URL + ' for details of checked urls.',
                      options);
  }
}
}  
/**
* Clears the summary sheet in the link checker spreadsheet.
*
* @param {Object} spreadsheet The link checker spreadsheet object.
*/
function clearSummarySheet(spreadsheet) {
  var summarySheet = getSummarySheet(spreadsheet);
  // Delete all the rows > 20.
  if (summarySheet.getMaxRows() > 20) {
    Logger.log('Deleting %s rows from 20', summarySheet.getMaxRows() - 20);
    summarySheet.deleteRows(20, summarySheet.getMaxRows() - 20);
  }
  // Clear row 20.
  if (summarySheet.getMaxRows() > 20) {
    summarySheet.getRange(20, 1, 1, 3).setValue('');
  }
}

/**
* Updates the summary sheet in the link checker spreadsheet.
*
* @param {Object} spreadsheet The link checker spreadsheet object.
*/
function updateCheckedUrlSummary(spreadsheet) {
  var settings = readSettings(spreadsheet);
  var accountSelector = getAccountSelector(settings).get();
  var totalAccounts = accountSelector.totalNumEntities();
  var summarySheet = getSummarySheet(spreadsheet);
  var totalBadUrls = 0;
  
  while (accountSelector.hasNext()) {
    var account = accountSelector.next();
    var sheet = getSheetForAccount(account, spreadsheet);
    var urlRange = sheet.getRange(4, 3, sheet.getMaxRows() - 3, 1).getValues();

    var goodUrls = 0;
    var badUrls = 0;

    for (var i = 0; i < urlRange.length; i++) {
      for (var j = 0; j < urlRange[i].length; j++) {
        if (!urlRange[i][j]) {
          continue;
        }

        if (parseInt(urlRange[i][j]) < 300) {
          goodUrls++;
        } else {
          badUrls++;
        }
      }
    }
    totalBadUrls = totalBadUrls + badUrls;

    summarySheet.appendRow(['', account.getCustomerId(), goodUrls, badUrls]);
    Logger.log('Account: %s, good urls: %s, bad urls: %s',
        account.getCustomerId(), goodUrls, badUrls);
  }

  return totalBadUrls;
}
