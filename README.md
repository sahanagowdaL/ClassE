/****
* Component Name: AICXClaimsPage
* Created By: Rahul Jain
* Purpose of Creation: To provide a customizable claims section for HAP Mobile Application.
* Date of Creation: 03-07-2024
* Modified Version: 1.0
* Last Modified By :
* Type: Lightning Web Component (LWC)
* Description:Component Is controlling Home Screen Activities 
* 
*/
import { LightningElement, api, wire, track } from 'lwc';
import { getRecord } from 'lightning/uiRecordApi';
import USER_ID from '@salesforce/user/Id';
const FEDERATION_IDENTIFIER_FIELD = 'User.FederationIdentifier';
import { OmniscriptBaseMixin } from 'vlocity_ins/omniscriptBaseMixin';
import { NavigationMixin } from 'lightning/navigation';
import orignalHap from '@salesforce/resourceUrl/haphomescreen';
import createTransactionLog from '@salesforce/apex/AICXTransactionLog.createTransactionLog';


export default class AICXClaimsPage extends OmniscriptBaseMixin(NavigationMixin(LightningElement)) {

    userId = USER_ID;
    _federationIdentifier;
    @track claims =[];
    @track claimsListSize;
    isClaimsNotFound = false;
    @track errorMessage = '';
    @track selectedStartDate = '';
    @track selectedEndDate = '';
    minDate;
    maxDate;
    data = [];
    optionslist = [];
    hasFetchedMemberDetails = false;
    @track value = '';
    isOver18 = false ;
    isUnder13 = false;
    isBetween13And18 = false;
    @track startDate;
    @track endDate;
    @track isSearchButtonDisabled = true;
    value = 'inProgress';
    Claims = orignalHap + '/images/Claims_icon_2.png';
    editCalendar = orignalHap + '/images/editCalendar.png';

    ipAddress;
    loggedInUserId = USER_ID;

    @track pageSize = 3; // Number of records per page
    @track currentPage = 1;
    @track pagedRecords = [];

    handlePageChange(event) {
        this.currentPage = event.detail.page;
        this.updatePagedRecords();
    }

    updatePagedRecords() {
        const start = (this.currentPage - 1) * this.pageSize;
        const end = start + this.pageSize;
        this.pagedRecords = this.claims.slice(start, end);
    }

    /*connectedCallback() {
        this.logTransaction();
		console.log('connected callback called== '+JSON.stringify(this.omniJsonData));
        if(this.omniJsonData?.data){
            this.startDate = this.omniJsonData.data;
            }  
         console.log('disconnectedCallback==conn=='+ this.startDate);   

    }*/

   /* connectedCallback() {
    // Log the transaction
    this.logTransaction();
    console.log('connected callback called== ' + JSON.stringify(this.omniJsonData));

    // Check sessionStorage for stored data
    const storedStartDate = sessionStorage.getItem('start_date');
    if (storedStartDate) {
        this.startDate = storedStartDate; // Use the stored value
    }

    // If data exists in omniJsonData, assign startDate to it
    if (this.omniJsonData?.data) {
        this.startDate = this.omniJsonData.data;
    }

    console.log('disconnectedCallback==conn== ' + this.startDate);
}

// To store the data when user changes startDate (or any other event)
handleStartDateChange(event) {
    this.selectedStartDate = event.detail.value;

    // Store the selected start date in sessionStorage
    sessionStorage.setItem('start_date', this.selectedStartDate);

    this.validateDateRange();
    console.log('this.selectedStartDate==' + this.selectedStartDate);
}*/

        setTosessionStorage() {
    sessionStorage.setItem('startDate', this.startDate);
    sessionStorage.setItem('endDate', this.endDate);
    sessionStorage.setItem('previousPageUrl', document.referrer); // Store previous page URL
}

getFromsessionStorage() {
    this.startDate = sessionStorage.getItem('startDate');
    this.endDate = sessionStorage.getItem('endDate');
}

connectedCallback() {
    if (document.getElementById('view-button')) {
        sessionStorage.setItem('previousPageUrl', document.referrer);
    } else {
        console.error('Error');
    }

    // Check if the captured time exists
    const sessionTime = sessionStorage.getItem('sessionTime');
    if (sessionTime) {
        const parsedSessionTime = new Date(sessionTime).getTime(); // Convert session time to milliseconds
        const currentTime = new Date().getTime(); // Get current timestamp in milliseconds
        const timeDifference = currentTime - parsedSessionTime; // Difference in milliseconds
        console.log('timeDifference === ' + timeDifference);

        // Check if session time is expired (greater than 15 minutes)
        if (!parsedSessionTime || timeDifference > 15 * 60 * 1000) {
            sessionStorage.removeItem('startDate');
            sessionStorage.removeItem('endDate');
            console.log('Session expired or invalid');
        } else {
            this.getFromsessionStorage();
            console.log('Session is within valid time');
        }
    } else {
        console.log('No capture time found in session storage');
    }
}

disconnectedCallback() {
    const currentDateTime = new Date();
    console.log('currentDateTime == ' + currentDateTime);
    sessionStorage.setItem('sessionTime', currentDateTime);
    this.setTosessionStorage();
    let previousPageUrl = sessionStorage.getItem('previousPageUrl');
    console.log(previousPageUrl);
}

logTransaction() {
    const transactionName = 'Claims';
    createTransactionLog({
        transactionName: transactionName,
        ipAddress: '',
        userId: this.loggedInUserId
    })
    .catch(error => {
        console.error('Error inserting transaction log:', error.body.message);
    });
}

// Wire Service to get current user record and federation identifier
@wire(getRecord, { recordId: '$userId', fields: [FEDERATION_IDENTIFIER_FIELD] })
wiredUser({ error, data }) {
    if (data) {
        this.federationIdentifier = data.fields.FederationIdentifier.value;
        this.loggedInUserHapMemberRecordNumber = this.federationIdentifier;            
    } else if (error) {
        console.error('Error retrieving user data: ', error);
    }
}
    // Getter for federation identifier
    get federationIdentifier() {
        return this._federationIdentifier;
    }

    // Setter for federation identifier with logic to fetch member details and initialize component
    set federationIdentifier(value) {
        if (value && value !== this._federationIdentifier) {
            this._federationIdentifier = value;
            this.isLoading = true;
    
            (async () => {
                try {
                    if (!this.hasFetchedMemberDetails) {
                        await this.getMemberPlanANDMemberDetails();
						if(!this.startDate && !this.endDate){
							this.setDefaultDates();
						}
                       
                        if (this._federationIdentifier && this.startDate && this.endDate) {
                            this.initializeComponent();
                        } else {
                            console.error('Mandatory values are missing:', {
                                federationIdentifier: this._federationIdentifier,
                                startDate: this.startDate,
                                endDate: this.endDate
                            });
                        }
                    }
                } catch (error) {
                    console.error('Error in fetching member plan and details:', error);
                } finally {
                    this.isLoading = false;
                }
            })();
        }
    }

    // Handle change in start date
    handleStartDateChange(event) {
        this.selectedStartDate = event.detail.value;        
        this.validateDateRange();
        console.log('this.selectedStartDate=='+ this.selectedStartDate);
    }

    // Handle change in end date
    handleEndDateChange(event) {
        this.selectedEndDate = event.detail.value;        
        this.validateDateRange();
         console.log('this.selectedEndDate=='+ this.selectedEndDate);
    }

    // Calculate the minimum date (5 years ago from today)
    calculateMinDate() {
        const today = new Date();
        const fiveYearsAgo = new Date(today.setFullYear(today.getFullYear() - 5));
        this.minDate = fiveYearsAgo.toISOString().split('T')[0];
    }

    // Calculate date ranges (min and max dates)
    calculateDateRanges() {
        const today = new Date();
        const fiveYearsAgo = new Date(today);
        fiveYearsAgo.setFullYear(today.getFullYear() - 5);

        this.minDate = fiveYearsAgo.toISOString().split('T')[0];
        this.maxDate = today.toISOString().split('T')[0];
    }

    // Set default dates and select the first member as default
    setDefaultDates() {
        const today = new Date();
        const threeMonthsAgo = new Date(today);
        threeMonthsAgo.setMonth(today.getMonth() - 3);

        this.startDate = threeMonthsAgo.toISOString().split('T')[0];
        this.endDate = today.toISOString().split('T')[0];
        if (this.options.length > 0) {
            const firstMember = JSON.parse(this.options[0].value);
            this.value = this.options[0].value;
            sessionStorage.setItem('Member', this.value);
            console.log('memberid' + this.value);

            this.federationIdentifier = firstMember.HAPMemberRecordNumber;    
            console.log('federationIdentifier');     
        } else {
            console.error('No options available for selection.');
        }
    }    

    // Initialize the component by fetching claims
    async initializeComponent() {        
        this.loading = true;
        const omnistudioStr = {
            memberRecordNumber: `"${this.federationIdentifier}"`,
            serviceDateFrom: `"${this.startDate}"`,
            serviceDateTo: `"${this.endDate}"`
        };        
        const params = {
            input: JSON.stringify(omnistudioStr),
            sClassName: 'vlocity_ins.IntegrationProcedureService',
            sMethodName: 'hapIP_AICX_ClaimResults',
            options: '{}',
        };
    
        try {
            const response = await this.omniRemoteCall(params, true);            
            const ipResult = response.result?.IPResult;            
            if (ipResult?.response && Array.isArray(ipResult.response) && ipResult.response[0]?.claims) {                
                this.claims = response.result.IPResult.response[0].claims;
                this.claims = this.claims.map(claim => {
                    if (claim.claimStatusDescription === 'In Progress') {
                        claim.viewDetailsButtonVisible = true;
                    } else {
                        claim.viewDetailsButtonVisible = false;
                    }
                    // Setting status flags based on claimStatusDescription
                    claim.claimStatusDescriptionIsComplete = claim.claimStatusDescription === 'Complete';
                    claim.claimStatusDescriptionIsDenied = claim.claimStatusDescription === 'Denied';
                    claim.claimStatusDescriptionIsAdjusted = claim.claimStatusDescription === 'Adjusted';
                    claim.claimStatusDescriptionIsInProgress = claim.claimStatusDescription === 'In Progress';

                    return claim;
                });
                if (this.isBetween13And18) {
                    this.claims = this.claims.filter(claim => claim.isSensitiveClaim !== 'Y');
                }

                this.claimsListSize = this.claims.length;
                this.updatePagedRecords();
                this.isClaimsNotFound = false;
                this.loading = false;
            } 
            else if (ipResult?.info?.status === 'No claims found' || ipResult?.success == false) {
                this.claims = [];
                this.claimsListSize = 0;
                this.isClaimsNotFound = true;
                this.loading = false;
            } 
            
            else {
                if (ipResult?.Name) {
                    // Handle error tracking with auto number
                    this.errorMessage = `Error Tracking ID: ${ipResult.Name}`;
                } else {
                    this.errorMessage = 'There is an issue with accessing your information at this time. Please try again later. Sorry for the inconvenience.';
                }
                    this.loading = false;
            }

        } catch (error) {
            console.log(JSON.stringify(error), ' inside error 2');
            this.showErrorModal = true;
            this.errorMessage = 'There is an issue with accessing your information at this time. Please try again later. Sorry for the inconvenience. Error Tracking ID :';
            this.loading = false;
        } finally {
            this.loading = false;
        }
    }
    
    // Handle date change event
   
    handleDateChange(event) {
        const { name, value } = event.target;   
        console.log('name==='+ name  + 'date value ==='+ value);     
        if (name === 'startDate') {
            this.startDate = value;
        } else if (name === 'endDate') {
            this.endDate = value;
        }
        this.validateDateRange();
        this.validateSearchButton();
       
		}
    // Validate the date range
    validateDateRange() {        
        let errorMessage = '';
        const today = new Date();
        const fiveYearsAgo = new Date(today);
        fiveYearsAgo.setFullYear(today.getFullYear() - 5);
        const startDate = new Date(this.startDate);
        const endDate = new Date(this.endDate);

        if (isNaN(startDate) || isNaN(endDate)) {
            errorMessage = 'Please select valid dates.';
        } else if (endDate > today || startDate > today) {
            errorMessage = 'The selected End Date must be on or before today';            
        } else if (startDate < fiveYearsAgo || endDate < fiveYearsAgo) {
            errorMessage = 'The selected Start Date must be within 5 years from today’s date.';
        } else if (Math.abs(endDate - startDate) > 365 * 24 * 60 * 60 * 1000) {
            errorMessage = 'The date range from the selected Start Date to the End Date must not exceed one year.';
        } else if (endDate < startDate) {
            errorMessage = 'The selected Start Date must be on or before the End Date.';
        }

        if (errorMessage) {
            this.errorMessage = errorMessage;
            this.isSearchButtonDisabled = true;
        } else {
            this.errorMessage = '';
            this.isSearchButtonDisabled = false;
        }
    }

    // Validate the search button
    validateSearchButton() {        
        const dropdown = this.template.querySelector('select');
        if (!dropdown.value) {
            this.isSearchButtonDisabled = true;
        } else {
            this.isSearchButtonDisabled = false;
        }
    }

    // Fetch member plans and details
    async getMemberPlanANDMemberDetails() {
        this.loading = true;
        this.responseReceived = false;        
        const omnistudioStr = {            
            recordId: `"${this.federationIdentifier}"`
        };
        const params = {
            input: JSON.stringify(omnistudioStr),
            sClassName: 'vlocity_ins.IntegrationProcedureService',
            sMethodName: 'hapIP_AICX_getAllRelatedMembers', 
            options: '{}',
        };
        await this.omniRemoteCall(params, true)
            .then(response => {
                try {
                    this.loading = false;
                    this.data = response.result.IPResult.memberPlanRecs.memberPlan;                    
                    const latestMemberPlan = this.getLatestMemberPlan(this.data);                    
                    const seenHAPMemberRecordNumbers = new Set();

                    // Filter and remove duplicates
                    const filteredPlans = this.data.filter(plan => {
                        const isUnique = !seenHAPMemberRecordNumbers.has(plan.HAPMemberRecordNumber);
                        const isValidPlan = plan.GroupId === latestMemberPlan.GroupId &&
                            plan.SubscriberId === latestMemberPlan.SubscriberId &&
                            (plan.RelationshipForBenefits === 'Primary' ||
                                plan.RelationshipForBenefits === 'Spouse' ||
                                plan.RelationshipForBenefits === 'Other Relationship' ||
                                plan.RelationshipForBenefits === 'Child');

                        if (isUnique && isValidPlan) {
                            seenHAPMemberRecordNumbers.add(plan.HAPMemberRecordNumber);
                            return true;
                        }

                        return false;
                    });                   
                    
                    /*let loginUserRecord = filteredPlans.find(a => a.HAPMemberRecordNumber === this.federationIdentifier);
                    if (loginUserRecord && loginUserRecord.RelationshipForBenefits != 'Primary') {
                        console.log('Its a dependent Login',JSON.stringify(loginUserRecord));
                        this.options = loginUserRecord.map(plan => {
                            return {
                                label: `${plan.FullName}`,
                                value: JSON.stringify(plan)
                            };
                        });
                    } else{
                        console.log('Its a primary Login');
                        this.options = filteredPlans.map(plan => {
                            return {
                                label: `${plan.FullName}`,
                                value: JSON.stringify(plan)
                            };
                        });
                    } */

                   let loginUserRecord = filteredPlans.find(a => a.HAPMemberRecordNumber === this.federationIdentifier);
                  //  if (loginUserRecord) {
                        if (loginUserRecord && loginUserRecord.RelationshipForBenefits != 'Primary') {
                            console.log('Its a dependent Login', JSON.stringify(loginUserRecord));
                            // If loginUserRecord is an object, you can't use map directly on it.
                            this.options = [{
                                label: `${loginUserRecord.FullName}`,
                                value: JSON.stringify(loginUserRecord)
                            }];
                        } else {
                            console.log('Its a primary Login');
                            this.options = filteredPlans.map(plan => {
                                return {
                                    label: `${plan.FullName}`,
                                    value: JSON.stringify(plan)
                                };
                            });
                        }
                  /*  } else {
                        console.log('No matching record found for the logged-in user.');
                        // Handle the case where no matching record is found
                        this.options = [];
                    } */

                    console.log('options---->336',JSON.stringify(filteredPlans));
                  
                    if(this.options.length > 0){
                        this.hasFetchedMemberDetails = true;
                    }                    
                } catch (error) {
                    console.log('some error' + error);
                }

            }).catch(error => {
                window.console.log(JSON.stringify(error), ' inside error ');
                this.loading = false;                
            });
    }

    // Navigate to claim details page
    getLatestMemberPlan(memberPlans) {        
        const convertDate = (dateString) => {
            const [month, day, year] = dateString.split('/');
            return `${year}${month.padStart(2, '0')}${day.padStart(2, '0')}`;
        };

        const getLatestPlanFromList = (plans) => {
            if (plans.length === 0) return null;
            return plans.reduce((latest, current) => {
                return convertDate(current.EffectiveFrom) > convertDate(latest.EffectiveFrom) ? current : latest;
            });
        };

        const activePlans = memberPlans.filter(plan => plan.MemberStatus === 'Active');
        if (activePlans.length > 0) {
            return getLatestPlanFromList(activePlans);
        }

        const pendingPlans = memberPlans.filter(plan => plan.MemberStatus === 'Pending');
        if (pendingPlans.length > 0) {
            return getLatestPlanFromList(pendingPlans);
        }

        const terminatedPlans = memberPlans.filter(plan => plan.MemberStatus === 'Terminated');
        if (terminatedPlans.length > 0) {
            return getLatestPlanFromList(terminatedPlans);
        }
        
        return null;
    }

    // Handle API call using Apex
    async handleDetailsClick(event) {
        const targetId = event.target.dataset.id;
        const stateParameters = {
            c__data: targetId
        };
        
        this[NavigationMixin.Navigate]({
            type: 'standard__webPage',
            attributes: {
                url: '/claims/claim-Details?c__data='+targetId 
            },
            state: stateParameters
        });
    }    

    /**
     * Handles the change event for the dropdown selection.
     * Extracts the selected value and updates the federation identifier.
     * Calculates the age based on the member's date of birth and sets various flags.
     * Displays an error message if the selected member is over 18, and clears claims.
     */
    handleChange(event) {

        this.value = event.detail.value;        
        const memberRecordNumber = JSON.parse(event.detail.value);
        console.log('memberRecordNumber=='+ memberRecordNumber);
        this.federationIdentifier = memberRecordNumber.HAPMemberRecordNumber;
        console.log('federationIdentifier=='+ this.federationIdentifier);
        const dob = new Date(memberRecordNumber.DOB);
        const age = this.calculateAge(dob);        
        if(this.loggedInUserHapMemberRecordNumber != this.federationIdentifier){
            this.isOver18 = age > 18;
            this.isUnder13 = age < 13;
            this.isBetween13And18 = age >= 13 && age < 18;
            if (this.isOver18) {
                this.errorMessage = 'You cannot view Claims for dependents over 18.';
                this.claims = [];
                this.claimsListSize = 0;
                this.isClaimsNotFound = true;
                this.loading = false;
            }else{
                this.errorMessage = '';
            }
        }else{
            this.isOver18 = false;
            this.isUnder13 = false;
            this.isBetween13And18 = false;
            this.errorMessage = '';
        }     
    }

    /**
     * Handles the search result event.
     * Validates the presence of required values and initializes the component.
     */
    handleSearchResult(event) {

        if (this.errorMessage) {
            this.error = this.errorMessage;
            return;
        }
    
        if (!this.value) {
            this.error = 'Please select a member';
            return;
        }
    
        if (!this.federationIdentifier) {
            console.error('Federation Identifier is missing.');
            return;
        }

        if (this.isOver18) {
            this.errorMessage = 'You cannot view Claims for dependents over 18.';
            return;
        }

        this.claims = [];
        this.claimsListSize = 0;
        this.isClaimsNotFound = true;
        this.pageSize = 3; 
        this.currentPage = 1;
        this.pagedRecords = [];
        this.updatePagedRecords();
    
        this.initializeComponent();
    }

    /**
     * Calculates the age based on the given date of birth.
     * @param {Date} dob - The date of birth.
     * @returns {number} The calculated age.
     */
    calculateAge(dob) {
        const today = new Date();
        let age = today.getFullYear() - dob.getFullYear();
        const monthDiff = today.getMonth() - dob.getMonth();
        if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < dob.getDate())) {
            age--;
        }        
        return age;
    }
}
