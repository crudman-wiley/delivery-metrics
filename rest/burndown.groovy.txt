import com.onresolve.scriptrunner.runner.rest.common.CustomEndpointDelegate
import groovy.transform.BaseScript
import javax.ws.rs.core.MultivaluedMap
import javax.ws.rs.core.Response
import groovy.xml.MarkupBuilder

@BaseScript CustomEndpointDelegate delegate

burndownB(httpMethod: "GET", groups: ["jira-users"]) { MultivaluedMap queryParams, String body ->
    BurndownReport report = new BurndownReport()
    def response = Response.ok( report.build() ,"text/html" )
    return response.build()
}

class BurndownReport {
  @groovy.transform.TypeChecked(groovy.transform.TypeCheckingMode.SKIP)
  public String build(){
    def useMultiSelect = false
    StringWriter writer = new StringWriter();
    def builder = new MarkupBuilder(writer);
    def mkp = builder.mkp;
    writer.append("<!DOCTYPE html>")
    def aspects = [ committed: "Committed Epics", uncommitted: "Other Epics", noEpics: "No Epics" ];
    builder.html{
      head {
        script(src:"https://cdn.jsdelivr.net/npm/d3@7","")
        script(src:"https://cdn.jsdelivr.net/npm/@observablehq/plot@0.6.14","")
        script(type:"text/javascript"){
          mkp.yieldUnescaped( getJavascript() );
        }
        style(type:"text/css"){
          mkp.yieldUnescaped( getCSS() );
        }
      }
      body(onload:"events.load()"){
        div(id:"dashboards", class:"horizontal-flex-container"){
          div(id:'layout-panel-left', class:'vertical-flex-container'){
            div(id:"summary-dashboard", class:"dashboard-panel"){
              table(id:"summary-table"){
                thead{
                  tr{
                    th("Planning Interval")
                    td{
                      select(id:"sprint-selector",onchange: "events.changeSprintSelector(this)","")
                    }
                  }
                  tr{
                    th("Portfolio")
                    td{
                      if (useMultiSelect == true){
                        select(id:"portfolio-selector",multiple: true, onchange: "events.changePortfolioFilter()", "")
                      } else {
                        select(id:"portfolio-selector", onchange: "events.changePortfolioFilter()", "")
                      }
                    }
                  }
                  tr{
                    th("Program")
                    td{
                      if (useMultiSelect == true){
                        select(id:"program-selector",multiple: true, onchange: "events.changeProgramFilter()","")
                      } else {
                        select(id:"program-selector",onchange: "events.changeProgramFilter()","")
                      }
                    }
                  }
                  tr{
                    th("Team")
                    td{
                      if (useMultiSelect == true){
                        select(id:"team-selector",multiple: true, onchange: "events.changeTeamFilter()","")
                      } else {
                        select(id:"team-selector", onchange: "events.changeTeamFilter()","")
                      }
                    }
                  }
                  tr{
                    th("State")
                    td(id:"sprint-status", class: "align-center dynacell","")
                  }
                  tr{
                    th("Start")
                    td(id:"sprint-start", class:"align-center dynacell","")
                  }
                  tr{
                    th("End")
                    td(id:"sprint-end", class: "align-center dynacell","")
                  }
                }
                tbody{
                    tr{
                        th("Completed")
                        td(id:"summary-completed",class:"numeric dynacell","")
                    }
                    tr{
                        th("Canceled")
                        td(id:"summary-canceled",class:"numeric dynacell","")
                    }
                    tr{
                        th("Incomplete")
                        td(id:"summary-incomplete",class:"numeric dynacell","")
                    }
                    tr(class:"subtotal"){
                        th("Subtotal")
                        td(id:"summary-subtotal",class:"numeric dynacell","")
                    }
                    tr{
                        th("Removed")
                        td(id:"summary-removed",class:"numeric dynacell","")
                    }     
                    tr{
                        th("Completed Outside")
                        td(id:"summary-completed-outside",class:"numeric dynacell","")
                    }
                    tr{
                        th("Canceled Outside")
                        td(id:"summary-canceled-outside",class:"numeric dynacell","")
                    }                
                    tr(class: "total"){
                        th("Total")
                        td(id:"summary-total",class:"numeric dynacell","")
                    }
                }
              }
            }
          }
          div(id:'layout-panel-center', class:'horizontal-flex-container'){
            div(id:"committed-dashboard",class:"dashboard-panel always-visible bordered vertical-flex-container", onclick:"events.clickDashboard('committed')"){
              div(class:"chart-container hidden"){
                div(id:'committed-dashboard-header', class:"dashboard-header"){
                  div('')
                  div{
                    div(class:'inline','Committed Epics and Work Items')
                    div(class:'inline tooltip','ⓘ'){
                      span(class:'tooltiptext','Epics committed to the quarter (using the "Planning Interval" field), and any issues those Epics may contain.')

                    }
                  }
                  div{
                    div(class:'kpis'){
                      div("Predictability:")
                      div(id:"kpi-predictability", class:"kpi", "")
                    }
                  }
                }
                div(id:"committed-burndown-chart", class:"dynacell chart always-visible", "")
                div(class:"legend"){
                  div(class:"legend-item"){
                    label(class: "switch"){
                      input(
                        id:"checkbox-committed-scope",
                        class:"red",
                        "data-color":"red",
                        "data-aspect":"committed",
                        type:"checkbox",
                        checked: true,
                        onchange:'events.changeCheckBox(this);',
                        '')
                      span(class:"slider round", '')
                    }
                    span(class:'legend-text', 'Scope')
                  }
                  div(class:"legend-item"){
                    label(class: "switch"){
                      input(
                        id:"checkbox-committed-epics",
                        class:"blue",
                        "data-color":"blue",
                        "data-aspect":"committed", 
                        type:"checkbox",
                        checked: true, 
                        onchange:'events.changeCheckBox(this);','')
                      span(class:"slider round", '')
                    }
                    span(class:'legend-text', 'Epics')
                  }
                  div(class:"legend-item"){
                    label(class: "switch"){
                      input(id:"checkbox-committed-stories",class:"green","data-color":"green","data-aspect":"committed", type:"checkbox",checked: true, onchange:'events.changeCheckBox(this);','')
                      span(class:"slider round", '')
                    }
                    span(class:'legend-text', 'Work Items')
                  }
                }
              }
              div(class:"loader",'')
            }
          }
          div(id:'layout-panel-right',class:'vertical-flex-container'){
            div(id:"uncommitted-dashboard",class:"dashboard-panel vertical-flex-container bordered always-visible", onclick:"events.clickDashboard('uncommitted')"){
              div(class:"chart-container hidden"){
                div(class:"dashboard-header"){
                  div(class:"inline","Other Epics and Work Items")
                  div(class:'inline tooltip','ⓘ'){
                    span(class:'tooltiptext','Epics that are not committed to the quarter but are experiencing some level of activity.')
                  }
                }
                div(id:"uncommitted-burndown-chart", class:"dynacell chart always-visible", "")
                div(class:"legend"){
                  div(class:"legend-item"){
                    label(class: "switch"){
                      input(id:"checkbox-uncommitted-epics",class:"blue","data-color": "blue", "data-aspect":"uncommitted", type:"checkbox",checked: true, onchange:'events.changeCheckBox(this);','')
                      span(class:"slider round", '')
                    }
                    span(class:'legend-text', 'Epics')
                  }
                  div(class:"legend-item"){
                    label(class: "switch"){
                      input(id:"checkbox-uncommitted-stories",class:"green","data-color":"green","data-aspect":"uncommitted", type:"checkbox",checked: true, onchange:'events.changeCheckBox(this);','')
                      span(class:"slider round", '')
                    }
                    span(class:'legend-text', 'Work Items')
                  }
                }              
              }
              div(class:"loader",'')
            }
            div(id:"noEpics-dashboard",class:"dashboard-panel vertical-flex-container bordered always-visible", onclick:"events.clickDashboard('noEpics')"){
              div(class:"chart-container hidden"){
                div(class:"dashboard-header"){
                  div(class:'inline','Work Items Without Epics')
                  div(class:'inline tooltip','ⓘ'){
                    span(class:'tooltiptext','Issues that are experiencing some level of activity but are not linked to an Epic.')
                  }
                }
                div(id:"noEpics-burndown-chart", class:"dynacell chart always-visible", "")
                div(class:"legend"){
                  div(class:"legend-item"){
                    label(class: "switch"){
                      input(
                        id:"checkbox-noEpics-stories",
                        class:"green",
                        "data-color":"green",
                        "data-aspect":"noEpics",
                        type:"checkbox",
                        checked: true,
                        disabled: true,
                        onchange:'events.changeCheckBox(this);',
                        '')
                      span(class:"slider round inoperable", '')
                    }
                    span(class:'legend-text', 'Work Items')
                  }
                }              
              }
              div(class:"loader",'')
            }
          }
        }
        div(id:"tables"){
          div(id:"issue-tables") {
            def sections = ["Completed","Canceled","Incomplete","Removed","Completed Outside","Canceled Outside"];
            sections.each{ section ->
              details(id:"${section.toLowerCase().replaceAll(" ","-")}", class: "hidden"){
                summary{
                    span(class:'subhead',section)
                    span(class:'section-count', '')
                }
                table{
                  thead{
                    tr{
                      th(class:'issue-type','T')
                      th(class: 'issue-key','Key')
                      th(class: 'issue-summary','Summary')
                      th(class: 'issue-status', 'Status')
                      th(class: 'issue-progress','Progress')
                    }
                  }
                  tbody('')
                }
              }
            }
          }
          div(id:"event-tables","")
        }
        div(id:'epic-panel', class:"epic-panel-hidden"){
          div(id:'epic-panel-header', class:'vertical-flex-container'){
            table(id:'epic-panel-header-summary'){
              tbody{
                tr{
                  th('Issue:')
                  td(id:'epic-panel-issue-key','')
                }
                tr{
                  th('Summary:')
                  td(id:'epic-panel-issue-summary',class:'issue-summary','')
                }
              }
            }
            div(id:'close',onclick:'events.clickCloseEpicPanel()','×')
          }          
          div(id:'epic-panel-body'){
            table(id:'epic-panel-story-table') {
              thead{
                th(class:'align-center', 'T')
                th('Key')
                th('Summary')
                th('Team')
                th('Status')
              }
              tbody('')
            }
          }
        }
      }
    }
    return writer.toString()
  }
  private String getJavascript(){
    return '''
    class Events{
      constructor(){}
      async clickDashboard( group ){
        if( group != view.active) {
          document.querySelector(".shadow").classList.remove("shadow")
          const sprint = model.sprints.find( it => it.id == Number(document.querySelector("#sprint-selector").value) );
          view.active = group;
          model.update(sprint, group);
        }
      }
      async load(){
        view.active = "committed";
        model.sprints = await jira.getSprintsForBoard(4554);
        const sprint = model.sprints.find( it => it.state == 'active' );
        view.buildSelector("#sprint-selector", model.sprints, sprint.id);
        const dictionaries = await jira.getDictionaries();
        Object.keys(dictionaries).forEach( k => model[k] = dictionaries[k]);
        
        model.update(sprint,"uncommitted");
        model.update(sprint, "noEpics");
        model.update(sprint, "committed");
      }
      changeCheckBox( checkbox ){ 
        const sprint = model.sprints.find( it => it.id == Number(document.querySelector("#sprint-selector").value) );
        const group = checkbox.getAttribute("data-aspect");
        const filters = view.getFilters();
        view.updateBurndownChart(sprint, group, filters);
      }
      changePortfolioFilter( ) {
        const sprint = model.sprints.find( it => it.id == Number(document.querySelector("#sprint-selector").value) );
        const filters = view.getFilters()
        view.buildFilter("#program-selector", model.getPrograms(sprint, view.active, filters.portfolio), "all");
        this.changeProgramFilter();
      }
      changeProgramFilter( ) {
        const sprint = model.sprints.find( it => it.id == Number(document.querySelector("#sprint-selector").value) );
        const filters = view.getFilters()
        view.buildFilter("#team-selector", model.getTeams(sprint, view.active, filters.portfolio, filters.program), "all");
        this.changeTeamFilter()
      }
      changeSprintSelector( option ){
        const sprint = model.sprints.find( it => it.id == Number(option.value) );
        view.active = "committed";
        view.reset();
        model.update(sprint,"committed");
        model.update(sprint,"uncommitted");
        model.update(sprint,"noEpics");
      
      }
      changeTeamFilter( ) {
        const sprint = model.sprints.find( it => it.id == Number(document.querySelector("#sprint-selector").value) );
        view.applyFilters(sprint);
      }
      clickCloseEpicPanel(){
        view.hideEpicPanel();
      }
      clickIssueKey(element){
        const sprint = model.sprints.find( it => it.id == Number(document.querySelector("#sprint-selector").value) );
        const key = element.innerText;
        const objectCache = sprint[view.active].epics ? sprint[view.active].epics : sprint[view.active].stories;
        const issue = Object.values( objectCache ).reduce( (collector, x) => collector.concat(x), new Array()).find( it => it.key == key);
        if ( !issue || issue.issueType != "Epic" ){
          window.open(`/browse/${key}`, "_blank")
          return new Array();
        }

        const summary = element.closest('tr').querySelector(".issue-summary").innerText;
        const subgroup = element.closest("details").id;

        const payload = {key: key, summary: summary, stories: issue.stories };
        element.classList.add('visited');
        view.hideEpicPanel();

        const tr = element.closest('tr');
        view.showEpicPanel( sprint, payload );
      }
    }

    class Jira {
      constructor(){}
      async getDictionaries(){
        const urls = {
          'issuetypes': '/rest/api/latest/issuetype',
          'statuses': '/rest/api/latest/status'
        };
        const keys = Object.keys(urls);
        const dictionaries = {};
        try {
          const fetchPromises = Object.values(urls).map( url => fetch(url) );
          const responses = await Promise.all(fetchPromises);
          const data = await Promise.all(responses.map(response => response.json()));
          for(let i = 0; i < keys.length; i++){
            const map = {};
            data[i].forEach((d) => {
              map[d.name] = d;
            });
            dictionaries[keys[i]] = map;
          }
          return dictionaries;
        } catch( error ){
          console.error( "Error fetching data:", error );
        }
      }
      async getBurndownData( sprint, group){
        if( sprint[group]?.events ){
          return false;
        }
        const urls = {
          epics: `/rest/scriptrunner/latest/custom/deliveryMetrics?sprint=${sprint.id}&group=${group}&type=epics&dataset=events`,
          stories: `/rest/scriptrunner/latest/custom/deliveryMetrics?sprint=${sprint.id}&group=${group}&type=stories&dataset=events`
        };
        const keys = Object.keys(urls);
        try{
          const fetchPromises = Object.values(urls).map( url => fetch(url) );
          const responses = await Promise.all(fetchPromises);
          const data = await Promise.all(responses.map(response => response.json()));
          if (!sprint[group]){
            sprint[group] = {};
          }
          sprint[group].events = {};
          sprint[group].events.epics = data[keys.indexOf("epics")];
          if ( group == "committed") {
            const openIssues = new Set();
            const issuesInScope = new Set();
            const scopeData = new Array();
            sprint[group].events.epics.forEach( (it) => {
              switch( it.type ){
                case "ISSUE_CREATED":
                  if (it.weight == 1){
                    it.scopeWeight = 1;
                  } else {
                    it.scopeWeight = 0;
                  }
                  openIssues.add(it.key);
                  break;
                case "ISSUE_COMPLETED":
                case "ISSUE_CANCELED":
                  if (!openIssues.has(it.key) ){
                    it.weight = 0;
                  }
                  openIssues.delete(it.key);
                  break;
                case "ISSUE_REOPENED":
                  if( openIssues.has(it.key) ){
                    it.weight = 0;
                  }
                  openIssues.add(it.key);
                  break;
                case "ISSUE_ADDED":
                  it.scopeWeight = 1;
                  if ( issuesInScope.has(it.key) ){
                    it.weight = 0;
                    it.scopeWeight = 0;
                  }
                  break;
                case "ISSUE_REMOVED":
                  it.scopeWeight = -1;
                  if ( !issuesInScope.has(it.key) ){
                    it.scopeWeight = 0;
                  }
              }
              if( new Date(it.created).getTime() < new Date(sprint.startDate).getTime() ){
                it.scopeWeight = it.weight;
              }
              if ( it.scopeWeight == 1){
                issuesInScope.add( it.key );
              } else if (it.scopeWeight == -1){
                issuesInScope.delete(it.key);
              }
              if( new Date(it.created).getTime() < new Date(sprint.startDate).getTime() || it.type == 'ISSUE_ADDED' || it.type == 'ISSUE_REMOVED'){
                scopeData.push( {
                  created: it.created,
                  id: it.id,
                  key: it.key,
                  portfolio: it.portfolio,
                  program: it.program,
                  sequence: it.sequence,
                  series: "Scope",
                  sprint: it.sprint,
                  type: it.type,
                  team: it.team,
                  weight: it.scopeWeight
                })
              }
            });
            sprint[group].events.epics = sprint[group].events.epics.concat(scopeData);
          }
          sprint[group].events.stories = data[keys.indexOf("stories")];
        } catch (error){
          sprint[group].events.epics = [];
          sprint[group].events.stories = [];
          console.error( "Error fetching data:", error );
        }
        return true;
      }
      async getInventory( sprint, group ){
        if( sprint[group]?.epics || sprint[group]?.stories ) {
          return false;
        }
        const urls = {};
        urls.stories = `/rest/scriptrunner/latest/custom/deliveryMetrics?sprint=${sprint.id}&group=${group}&type=stories&dataset=states`;
        if ( group != "noEpics"){
          urls.epics = `/rest/scriptrunner/latest/custom/deliveryMetrics?sprint=${sprint.id}&group=${group}&type=epics&dataset=states`;
        }      
        const keys = Object.keys(urls);
        try{
          const fetchPromises = Object.values(urls).map( url => fetch(url) );
          const responses = await Promise.all(fetchPromises);
          const data = await Promise.all(responses.map(response => response.json()));
          if ( !sprint[group] ) {
            sprint[group] = {};
          }
          keys.forEach( (key) => {
            sprint[group][key] = {
              "completed":         data[keys.indexOf(key)].filter(it => it.outcome == "COMPLETED"),
              "canceled":          data[keys.indexOf(key)].filter(it => it.outcome == "CANCELED"),
              "incomplete":        data[keys.indexOf(key)].filter(it => it.outcome == "NOT_STARTED" || it.outcome == "IN_PROGRESS"),
              "removed":           data[keys.indexOf(key)].filter(it => it.outcome == "REMOVED"),
              "completed-outside": data[keys.indexOf(key)].filter(it => it.outcome == "COMPLETED_OUTSIDE"),
              "canceled-outside":  data[keys.indexOf(key)].filter(it => it.outcome == "CANCELED_OUTSIDE")
            };
          });
          if ( group == 'committed' && sprint[group].epics ){
            sprint.committed.epics.completed = sprint.committed.epics.completed.filter(it => it.sprint);
            sprint.committed.epics.canceled = sprint.committed.epics.canceled.filter(it => it.sprint);
            sprint.committed.epics.incomplete = sprint.committed.epics.incomplete.filter(it => it.sprint);
          }         
        } catch (error){
          sprint[group].epics = [];
          sprint[group].stories = [];
          console.error( "Error fetching data:", error );
        }
        return true;
      }
      async getSprintsForBoard( rapidView ){
        try {
          const response = await fetch(`/rest/agile/1.0/board/${rapidView}/sprint?state=active,closed`);
          const json = await response.json();
          return json.values.filter(it => it.originBoardId == rapidView);
        } catch( error ){
          console.error( "Error fetching data:", error );
        }
      }
    }

    class Model {
      async update(sprint, group ){
        view.showLoader(group);
        await jira.getBurndownData(sprint, group);
        if (view.active == group){
          await jira.getInventory( sprint, group );
          view.initialize( sprint, group );
        }
        view.updateBurndownChart(sprint, group, view.getFilters())
        view.hideLoader(group);
      }
      applyFilters( dataset, filters ){
        return dataset?.filter( (it) => {
          const p1 = filters.portfolio, p2 = filters.program, p3 = filters.team;
          const t1 = it.portfolio, t2 = it.program, t3 = it.team

          return ( p1 == "all" || p1 == "none" && ( t1 == null || t1 == "" ) || p1 == t1) &&
            ( p2 == "all" || p2 == "none" && ( t2 == null || t2 == "" ) || p2 == t2) &&
            ( p3 == "all" || p3 == "none" && ( t3 == null || t3 == "" ) || p3 == t3);
        });
      }
      getDataset(sprint,group){
        return sprint[group].epics;
      }
      getPortfolios(sprint, group){
        const dataset = Object.values( sprint[group].epics ? sprint[group].epics : sprint[group].stories ).reduce( (collector, it) => collector.concat( it ), new Array());
        const portfolios = [...new Set(dataset.flatMap(it => it.portfolio ))].sort();
        return portfolios;
      }
      getPrograms( sprint, group, portfolio ){
        const dataset = Object.values(sprint[group].epics ? sprint[group].epics : sprint[group].stories  ).reduce( (collector, it) => collector.concat( it ), new Array());
        return [...new Set(dataset.filter( (it) => {
            switch( portfolio ){
              case "all": return true;
              case "none": return (it.portfolio == null || it.portfolio == "");
              default: return portfolio == it.portfolio;
            }
        }).flatMap(it => it.program))].sort();
      }
      getTeams( sprint, group, portfolio, program ){
        const dataset = Object.values(sprint[group].epics ? sprint[group].epics : sprint[group].stories  ).reduce( (collector, it) => collector.concat( it ), new Array());
        return [...new Set(dataset.filter( (it) => {
            switch( portfolio ){
              case "all": return true;
              case "none": return (it.portfolio == null || it.portfolio == "");
              default: return portfolio == it.portfolio;
            }
        }).filter( (it) => {
            switch( program ){
              case "all": return true;
              case "none": return (it.program == null || it.program == "");
              default: return program == it.program; 
            }
        }).flatMap(it => it.team))].sort();
      }
    }

    class View {
      constructor(){}
      applyFilters(sprint){
        const filters = this.getFilters();
        const inventory = sprint[view.active].epics ? sprint[view.active].epics : sprint[view.active].stories;
        this.updateSummaryTable(sprint, inventory, filters);
        this.updateSections(inventory, filters);
        this.updateBurndownChart(sprint,"committed", filters);  
        this.updateBurndownChart(sprint,"uncommitted", filters);  
        this.updateBurndownChart(sprint,"noEpics", filters);  
      }
      buildFilter( selector, values, selection ){
        const select = document.querySelector(selector);
        this.clear(select);
        const options = ["all","none"].concat(values).filter( it => it);
        for(const d of options){
          const option = document.createElement('option');
          option.setAttribute('value', d);
          option.innerText = d;
          if (d == selection){
            option.selected = true;
          }
          select.appendChild(option);
        }

      }
      buildIssueTableRow( obj ){
        const tr = document.createElement("tr");
        tr.dataset.key = obj.key;
        tr.appendChild( this.buildTableCellIssueType(obj) );
        tr.appendChild( this.buildTableCellIssueKeyPseudoLink(obj) );
        tr.appendChild( this.buildTableCellIssueSummary(obj) );
        tr.appendChild( this.buildTableCellIssueStatus(obj) );
        if (view.active != "noEpics"){
          tr.appendChild( this.buildTableCellIssueProgress(obj) );
        }
        return tr;
      }
      buildSelector(selector, options, selection){
        const select = document.querySelector(selector);
        for(const d of options){
          const option = document.createElement('option');
          option.setAttribute('value', d.id);
          option.innerText = d.name;
          if (d.id == selection){
            option.selected = true;
          }
          select.appendChild(option);
        }
      }
      buildTableCellIssueType(obj){
        const td = document.createElement("td")
        td.classList.add("issue-type");
        const img = document.createElement("img");
        img.setAttribute("src", model.issuetypes[obj.issueType].iconUrl);
        img.setAttribute("title",obj.issueType);
        td.appendChild(img);
        return td;
      }
      buildTableCellIssueKey(obj){
        const td = document.createElement("td")
        td.classList.add("issue-key")
        const a = document.createElement("a");
        a.innerText = obj.key;
        a.setAttribute("href", `/browse/${obj.key}`);
        a.setAttribute("target","_blank");
        td.appendChild(a);
        return td;
      }
      buildTableCellIssueKeyPseudoLink(obj){
        const td = document.createElement("td")
        td.classList.add("issue-key")
        const span = document.createElement("span");
        span.classList.add("pseudolink");
        span.setAttribute("onclick","events.clickIssueKey(this)");
        span.innerText = obj.key;
        td.appendChild(span);
        return td;
      }
      buildTableCellIssueSummary(obj){
        const td = document.createElement("td");
        td.classList.add("issue-summary");
        td.innerText = obj.summary;
        return td;
      }
      buildTableCellIssueStatus(obj){
        //<div class="lozenge color-success">Closed</div>
        const td = document.createElement("td")
        td.classList.add("issue-status");
        const div = document.createElement("div");
        const statusObject = model.statuses[obj.status];
        div.classList.add("lozenge",`color-${statusObject?.statusCategory?.colorName}`);
        div.innerText = obj.status;
        td.appendChild(div)
        return td;
      }
      buildTableCellIssueTeam(obj){
        const td = document.createElement("td");
        td.classList.add("issue-team");
        td.innerText = obj.team;
        return td;
      }
      buildTableCellIssueProgress(obj){
        const td = document.createElement("td")
        td.classList.add("issue-progress");

        const progressBar = document.createElement("div");
        progressBar.classList.add("progress-bar");
        td.appendChild(progressBar);

        const inventory = obj.progress;
        const total = Object.values(inventory).reduce( (collector, x) => collector.concat(x), new Array()).length - inventory.deleted.length;
        const lobes = [];
        ["canceled","completed","in_progress","not_started"].forEach((k) =>{
          const items = inventory[k];
          if (items.length > 0){
            const div = document.createElement('div');
            div.classList.add(`progress-${k}`);
            div.style.width = `${100 * (items.length/total)}%`;

            const a = document.createElement("a");
            a.setAttribute("target", "_blank");
            a.setAttribute("href", `/issues/?jql=issue+in+(${[...items].toString()})`);
            a.innerText = items.length;
            div.appendChild(a);
            lobes.push(div);
          }
        });
        if (lobes.length > 0){
          lobes[0].classList.add('left-side');
          lobes[lobes.length - 1].classList.add('right-side');
        }
        lobes.forEach( l => progressBar.appendChild(l) );
        return td;
      }
      clear( element ){
        while(element?.firstChild){
          element.removeChild(element.firstChild);
        }
      }
      compareIssues(a,b){
        return view.compareIssueKeys( a.key, b.key);
      }
      compareIssueKeys( a, b  ){
        const A = a.split("-");
        const B = b.split("-");
        if ( A[0].localeCompare(B[0]) == 0){
          return Number(A[1]) - Number(B[1]);
        }
        return A[0].localeCompare(B[0]);
      }
      getFilters(){
        const portfolio = document.querySelector("#portfolio-selector").value
        const program = document.querySelector("#program-selector").value
        const team = document.querySelector("#team-selector").value
        return { 
          portfolio: ( portfolio == null || portfolio == '') ? 'all' : portfolio, 
          program: ( program == null || program == '') ? 'all' : program, 
          team: ( team == null || team == '') ? 'all' : team, 
        }
      }
      hideEpicPanel(){
        const panel = document.querySelector("#epic-panel");
        panel.classList.add("epic-panel-hidden");
        panel.classList.remove("epic-panel-revealed");
      }
      hideLoader(group){
        const widget = document.querySelector(`#${group}-dashboard`);
        widget.style.backgroundColor = null;
        const container = widget.querySelector(".chart-container");
        container.classList.remove("hidden");
        widget.querySelector(".loader").classList.add("hidden");
      }
      initialize( sprint, group ){
        const filters = this.getFilters();
        const portfolios = model.getPortfolios(sprint, group);
        const programs = model.getPrograms(sprint, group, filters.portfolio);
        const teams = model.getTeams(sprint,group,filters.portfolio, filters.program);
        this.buildFilter("#portfolio-selector", portfolios, filters.portfolio);
        this.buildFilter("#program-selector", programs, filters.program);
        this.buildFilter("#team-selector", teams, filters.team);
        this.applyFilters(sprint);
        document.querySelector(`#${group}-dashboard`).classList.add("shadow");      
      }
      reset(){
        document.querySelectorAll(".shadow").forEach( e => e.classList.remove("shadow") );
        document.querySelectorAll(".dynacell").forEach(e => this.clear(e));
        document.querySelectorAll("details").forEach(e => e.classList.add("hidden"));
      }
      showEpicPanel( sprint, payload ){
        const panel = document.querySelector("#epic-panel");
        panel.classList.remove("epic-panel-hidden");
        panel.classList.add("epic-panel-revealed");

        // build the panel header... issue key
        if (true){
          const td = document.querySelector("#epic-panel-issue-key");
          this.clear(td);
          const a = document.createElement("a");
          a.innerText = payload.key;
          a.setAttribute("href", `/browse/${payload.key}`);
          a.setAttribute("target","_blank");
          td.appendChild(a);
        }
        // build the panel header... issue summary
        if (true){
          const td = document.querySelector("#epic-panel-issue-summary");
          this.clear(td);
          td.innerText = payload.summary;
        }
        // build the panel table
        if (true){
          const tbody = document.querySelector("#epic-panel-body tbody");
          this.clear(tbody);

          payload.stories?.sort(this.compareIssueKeys).forEach( (s) => {
            const story = Object.values( sprint[view.active].stories ).reduce( (collector,x) => collector.concat(x), new Array() ).find( it => it.key == s);
            if (story){
              const tr = document.createElement('tr');
              tbody.appendChild(tr);
              tr.appendChild( this.buildTableCellIssueType(story));
              tr.appendChild( this.buildTableCellIssueKey(story) );
              tr.appendChild( this.buildTableCellIssueSummary(story) );
              tr.appendChild( this.buildTableCellIssueTeam(story) );
              tr.appendChild( this.buildTableCellIssueStatus(story) );
            }
          });
        }
      }
      showLoader(group){
        const widget = document.querySelector(`#${group}-dashboard`);
        widget.style.backgroundColor = 'dimgray';
        const container = widget.querySelector(".chart-container");
        container.classList.add("hidden");
        widget.querySelector(".loader").classList.remove("hidden");
      }
      updateBurndownChart(  sprint, group, filters){
        const chart = document.querySelector(`#${group}-burndown-chart`);
        this.clear(chart);
        
        
                
        let data = model.applyFilters( sprint[group].events.epics, filters );
        if( sprint[group].epics ){
          const epics = Object.entries( sprint.committed.epics ).filter( it => ['completed','canceled','incomplete','removed'].includes(it[0])).reduce( (arr, x) => arr.concat(x[1]), new Array() )
          const issueKeys = new Set( [...epics.map(it => it.key)] );
          data = data.filter( it => issueKeys.has( it.key ) )
        }
        data = data.concat( model.applyFilters( sprint[group].events.stories, filters ) );   
        const switches = {
          epics: document.querySelector(`#checkbox-${group}-epics`)?.checked ? true : false,
          stories: document.querySelector(`#checkbox-${group}-stories`)?.checked ? true : false,
          scope: document.querySelector(`#checkbox-${group}-scope`)?.checked ? true : false,
        }
        if( switches.epics == false ){
          data = data.filter( it => it.series != "Epic")
        }
        if( switches.stories == false ){
          data = data.filter( it => it.series != "Story")
        }
        if(switches.scope == false ){
          data = data.filter(it => it.series != "Scope")
        }
        
        const accumulator = {};
        const temp = {};
        const last = {};

        const start = new Date(sprint.startDate).getTime();
        const end = new Date(sprint.endDate).getTime();
        const now = Date.now()
        const asOf = now > start && now < end ? now : end;
        const seriesLabels = {
          epic: "Epics",
          story: "Stories",
          scope: "Scope"
        }
        const comments = {
          'ISSUE_ADDED': "was added",
          'ISSUE_CANCELED': "was canceled", 
          'ISSUE_COMPLETED': "was completed", 
          'ISSUE_REMOVED': "was removed", 
          'ISSUE_REOPENED': "was reopened",
          'ISSUE_CREATED': "was created"
        }
        
        const burndown = []
        data.forEach( (it) => {
          if( !accumulator[it.series.toLowerCase()] ){
            accumulator[it.series.toLowerCase()] = 0;
          }
          const d = new Date(it.created).getTime();
          let w = it.weight;
          accumulator[it.series.toLowerCase()] = accumulator[it.series.toLowerCase()] + w;
          const m = { 
            type: seriesLabels[it.series.toLowerCase()], 
            date: d, 
            remaining: accumulator[it.series.toLowerCase()],
            note: `${it.key} ${comments[it.type]}`
          };
          if( m.date < start){
            temp[it.series.toLowerCase()] = m;
          }
          if( m.date >= start && m.date < end ){ 
            burndown.push(m); 
          }
          last[it.series.toLowerCase()] = m;
        });

        const baseline = temp["epic"] ? temp["epic"].remaining : temp["story"].remaining
        Object.keys(accumulator).forEach( (k) => {
          burndown.splice(0,0, {
            type: seriesLabels[k],
            date: new Date(sprint.startDate).getTime(), 
            remaining: temp[k] ? temp[k].remaining : 0,
            note: 'Sprint started' }
          );
          burndown.push({
            type: seriesLabels[k], 
            date: asOf, 
            remaining: last[k] ? last[k].remaining : 0,
            note: '' } );
        });

        const options = {
          width: 800,
          height: 290,
          marginTop: 20,
          marginBottom: 40,
          marginLeft: 60,
          color: { domain: ["Epics","Stories","Scope"], range: ["blue", "green", "red"] },
          x: {
            type: "time",
            interval: "hour",
            label: null, 
            domain: [ start, end ]
          },
          y:{ axis: "left", label: null },
          marks: [
            Plot.axisX( { ticks: "week" } ),
            Plot.gridX( { interval: "week" } ),
            Plot.ruleY([0]),
            Plot.lineY( burndown, { x: "date", y: "remaining", curve: "step", stroke: "type" } )
          ]
        };
        if ( now > start && now < end ){
          options.marks.push( 
            Plot.ruleX( [0], { x: now, stroke: "red", strokeWidth: 2, strokeOpacity: 0.2 } ) 
          );
        }
        options.marks.push(
            Plot.ruleY( [0], { y: baseline, stroke: "blue", strokeDasharray: 3, strokeWidth: 2, strokeOpacity: 0.2 } ),
            Plot.tip( burndown, Plot.pointerX( {
              x: "date", 
              y: "remaining", 
              title: (d) => [ `Date: ${d3.utcFormat("%Y-%m-%d")(d.date)}`, `${d.type}: ${d.remaining}`, `Comment: ${d.note}`].join("\\n"),
              filter: (d) => d.date < new Date() 
            }))
        );

        const plot = Plot.plot(options);
        chart.append(plot);
      }
      updateIssueTable( tbody, data ){
        this.clear(tbody);
        if ( view.active == "noEpics"){
          tbody.closest("table").querySelector(".issue-progress").classList.add("hidden");
        } else {
          tbody.closest("table").querySelector(".issue-progress").classList.remove("hidden");
        }
        data.sort(this.compareIssues).forEach( d => tbody.appendChild( this.buildIssueTableRow(d) ) );
      }
      updateSections(inventory, filters){
        [...document.querySelectorAll('details tbody')].forEach( it => this.clear(it));
        Object.keys(inventory).forEach( (k) => {
          const details = document.querySelector(`#${k}`);
          const selectedInventory = model.applyFilters( inventory[k] , filters);
          if( selectedInventory.length == 0 ){
            details.classList.add('hidden');
          } else {
            details.classList.remove('hidden');
            details.querySelector(".section-count").innerText = `(${selectedInventory.length})`;
            this.updateIssueTable( details.querySelector('tbody'), selectedInventory);
          }
        });
      }
      updateSummaryTable( sprint, inventory, filters ){
        document.querySelector("#sprint-status").innerText = sprint.state;
        document.querySelector("#sprint-start").innerText = new Date(sprint.startDate).toLocaleDateString();
        document.querySelector("#sprint-end").innerText = new Date(sprint.endDate).toLocaleDateString();

        const m = {
          completed: model.applyFilters( inventory["completed"], filters ).length,
          canceled: model.applyFilters( inventory["canceled"], filters ).length,
          incomplete: model.applyFilters( inventory["incomplete"], filters).length,
          removed: model.applyFilters( inventory["removed"], filters).length,
          completedOutside: model.applyFilters( inventory["completed-outside"], filters) .length,
          canceledOutside: model.applyFilters( inventory["canceled-outside"], filters).length,
          subtotal: function(){ return this.completed + this.canceled + this.incomplete },
          total: function(){ return this.subtotal() + this.removed + this.completedOutside + this.canceledOutside }
        }

        document.querySelector("#summary-completed").innerText = m.completed;
        document.querySelector("#summary-canceled").innerText = m.canceled;
        document.querySelector("#summary-incomplete").innerText = m.incomplete;
        document.querySelector("#summary-incomplete").innerText = m.incomplete;
        document.querySelector("#summary-subtotal").innerText = m.subtotal();
        document.querySelector("#summary-removed").innerText = m.removed;
        document.querySelector("#summary-completed-outside").innerText = m.completedOutside;
        document.querySelector("#summary-canceled-outside").innerText = m.canceledOutside;
        document.querySelector("#summary-total").innerText = m.total();
        
      }
    }

    const events = new Events();
    const jira = new Jira();
    const model = new Model();
    const view = new View();
    '''
  }

  private String getCSS(){
    return '''
      *{ font-family: Arial, Helvetica, sans-serif; font-size: small; }

      a { white-space: nowrap; font-size: inherit; text-decoration: none;}
      a:visited { color: #800080; }
      a:hover { text-decoration: underline; }
      a:active {color: red; }

      details { margin-top: 20px;}
      details table { margin-top: 10px; }
      h1 { display: inline; }
      input:active {
        cursor: wait;
      }
      input:checked + .slider:before {
        -webkit-transform: translateX(12px);
        -ms-transform: translateX(12px);
        transform: translateX(12px);
      }
      table{  width: 100%; border-collapse: collapse;}
      tbody tr:hover{ background: cornsilk; color: black; }
      td { vertical-align: top; }
      th{ text-align: left;}
      th,td { 
        border: 1px solid silver;
        padding: 4px;
        white-space: nowrap;
      }
      th:has(.sort-icon) { }
      th:has(.sort-icon):hover { background: #555; color: white; }
      select {
          max-width: 120px !important;
          overflow: hidden;
      }
      option {
          max-width: 100px !important;
          overflow: hidden;
      }      
      #chart { margin: 10px;}
      #close{ 
        position: absolute;
        top: 0;
        right: 0;
        font-size: 36px; 
        color: #ccc;
        margin-right: 10px;
      }
      #close:hover { color: white; }
      #close:active { color: #333; }
      #committed-dashboard-header{
        display: grid;
        grid-template-columns: 135px auto 135px;
      }
      #dashboards { 
        width: 100%;
      }
      #epic-panel {
        position: fixed;
        top: 0;
        border-radius: 10px;
        overflow-x: hidden;
        max-height: 80%;
        max-width: 70%;
      }
      #epic-panel>div { padding: 10px 10px 0px 10px; }
      #epic-panel-body-header {
        width: 100%;
      }

      #epic-panel-body-header {
        background: white;
        color: black;
        border: 1px solid silver;
      }

      #epic-panel-header { 
        position: sticky;
        top: 0;
        background: #555; 
        justify-content: space-between; 
      }
      #epic-panel-header-summary { width: fit-content; }
      #epic-panel-header-summary th { border: none; color: white; font-size: larger; }
      #epic-panel-header-summary td { border: none; color: white; font-size: larger; }
      #epic-panel-header tr:hover { background: none; }
      #epic-panel-header a:link {  color: white; }
      #epic-panel-header a:visited { color: #ddd; }
      #epic-panel-header a:hover { text-decoration: underline; }
      #epic-panel-header a:active { color: white; }
      #epic-panel-story-table { margin-bottom: 20px; }

      #issue-tables { margin-left: 6px; margin-right: 6px; }

      #layout-panel-left{ flex: 34;}
      #layout-panel-center{ flex: 113; margin-left: 8px;}
      #layout-panel-right{ 
        flex: 48; 
        transform: translateX(-3px); 
        justify-content: space-between;
        margin-right: 8px;
      }

      #legend-label {
        display: inline;
        font-size: x-small;
        vertical-align: top;
        font-weight: bold;
        padding-right: 5px;
      }
      #page-loader {
        transform: scale(3) translateX(-50%) translateY(100%);
        position: absolute;
        top: 0;
        left: 50%;
      }

      #sprint-status {text-transform: uppercase;}
      #summary-dashboard{}    
      #summary-table { }

      #uncommitted-dashboard { height: 50%; }
      #noEpics-dashboard { height: 50%; }
      [id$='total'] { font-weight: bold;}
      
      .align-center{ text-align: center;}
      .align-left{ text-align: left;}
      .align-right{ text-align: right;}
      .always-visible {}

      .blue:checked + .slider {
        background-color: blue;
      }

      .blue:focus + .slider {
        box-shadow: 0 0 1px blue;
      }
      .bordered {
        border: 1px solid silver;
      }

      .chart { 
        padding: 6px;
      }
      .chart-container{
        position: relative;
      }
      .clickable {
        cursor: pointer;
      }
      .clickable:hover:after{
        content: "▼";
      }
      .color-default { background-color: #dfe1e5; border-color: #dfe1e5; color: #42526e; }
      .color-inprogress { background-color: #deebff; border-color: #deebff; color: #0747a6; }
      .color-success{ background-color: #e3fcef; border-color: #e3fcef; color: #064; }
      .color-undefined{ background-color: orange; }
      .control-group {
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
      }
      .control {}

      .dashboard-panel {
        width: 100%;
        margin: 6px;
        position: relative;
      }
    
      .dashboard-header { 
        text-align: center;
        padding: 5px;
        font-weight: 600;
      }
      .data-filter {}
      .dynacell{}

      .epic-panel-issue-key a{ text-decoration: none; color: white; }
      .epic-panel-hidden {
        transition: 0.5s;
        right: -100%;
      }
      .epic-panel-revealed { 
        transition: 0.5s;
        right: 0;
        background-color: #ffffff;
        z-index: 2;
        box-shadow: 0px 0px 40px 5px #555555;
        transform: translate( -25%, 10px );
      }  
      .green:checked + .slider {
        background-color: green;
      }

      .green:focus + .slider {
        box-shadow: 0 0 1px green;
      }
      .red:checked + .slider {
        background-color: red;
      }

      .red:focus + .slider {
        box-shadow: 0 0 1px red;
      }
      .greyed-out { opacity: .3; transition: opacity .5s; }

      .hidden{ display: none; }
      .horizontal-flex-container { 
        display: flex; 
        flex-direction: row;
      }
      .inline {
        display: inline;
      }
      .inoperable::before{
        opacity: 0;
      }
      .inventory-item {
        
      }
      .issue-key { width: 150px; text-align: left; }
      .issue-progress { width: 150px; text-align: left; }
      .issue-summary{ white-space: normal; text-align: left;}
      .issue-status { width: 150px; text-align: left;}
      .issue-team { width: 150px; text-align: left; }
      .issue-type { width: 36px; text-align: center; }

      .kpi {
        padding-left: .5em;
      }
      .kpis {
        position: absolute;
        right: 0;
        margin-right: 5px;
        display: none;
        font-weight: normal;
      }
      
      .loader {
        top: 50%;
        left: 50%;
        width: 16px;
        height: 16px;
        border-radius: 50%;
        background-color: #fff;
        box-shadow: 32px 0 #fff, -32px 0 #fff;
        position: absolute;
        animation: flash 0.5s ease-out infinite alternate;
      }

    
      .label {
        font-weight: bold;
      }
      .left-side {border-top-left-radius: 6px; border-bottom-left-radius: 6px; }
      .legend{
        display: flex;
        align-items: center;
        justify-content: center;
        margin-bottom: 4px;
      }
      .legend-item{
        display: flex;
        align-items: center;
      }
      .legend-text{
        font-weight: bold;
        font-size: x-small;
        margin: 0px 2px 0px 2px;
      }
      .lozenge{
        display: inline;
        border: 1px solid;
        border-radius: 3px;
        padding: 2px;
        text-transform: uppercase;
        font-weight: 700;
        font-size: x-small;
      }

      .mini-loader { transform: scale(.5); position: relative; }

      .no-border { border: none; }
      .numeric { text-align: right; }

      .progress-bar {
        text-align: center; 
        height: 12px;
        line-height: 12px;
        vertical-align: middle;
        width: 100%;
        font-weight: normal;
      }
      .progress-bar * { font-size: 8px; }
      .progress-bar div { float: left; }
      
      .progress-canceled { color: white; background-color: #324259;}
      .progress-canceled a{ text-decoration: none; }
      .progress-canceled a:link { color: white; }
      .progress-canceled a:visited { color: white; }
      .progress-canceled a:hover { color: white; }
      .progress-canceled a:active { color: white; }

      .progress-completed { color: white; background-color: #526C92; }
      .progress-completed a{ text-decoration: none; }
      .progress-completed a:link { color: white; }
      .progress-completed a:visited { color: white; }
      .progress-completed a:hover { color: white; text-decoration: underline; }
      .progress-completed a:active { color: white; }

      .progress-in_progress { color: white; background-color: #8798B0; }
      .progress-in_progress a{ text-decoration: none;}
      .progress-in_progress a:link { color: white; }
      .progress-in_progress a:visited { color: white; }
      .progress-in_progress a:hover { color: white; }
      .progress-in_progress a:active { color: white; }

      .progress-not_started { color: black; background-color: #C1CBD9; }
      .progress-not_started a{ text-decoration: none; }
      .progress-not_started a:link { color: black; }
      .progress-not_started a:visited { color: black; }
      .progress-not_started a:hover { color: black; }
      .progress-not_started a:active { color: black; }

      .pseudolink{  color: blue; cursor: pointer;}
      .pseudolink:hover { text-decoration: underline; }
      .pseudolink:active { color: red; }

      .right-side {border-top-right-radius: 6px; border-bottom-right-radius: 6px; }

      .slider {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: #ccc;
        -webkit-transition: .2s;
        transition: .2s;
      }

      .slider:before {
        position: absolute;
        content: "";
        height: 12px;
        width: 12px;
        left: 2px;
        bottom: 2px;
        background-color: white;
        -webkit-transition: .2s;
        transition: .2s;
      }

      /* Rounded sliders */
      .slider.round {
        border-radius: 34px;
      }

      .slider.round:before {
        border-radius: 50%;
      }
      .shadow { box-shadow: 0px 0px 6px 3px #79e888; }
      .sort-icon { float: right; }
      .sorted:after {
        content: "▼";
      }
      .stealthy { color: white; }
      
      .subhead { font-weight: bold; }
      .subtotal { background: #ddd; }
      
      .switch {
        position: relative;
        width: 28px;
        height: 16px;
        margin: 2px;
      }

      .switch input { 
        opacity: 0;
        width: 0;
        height: 0;
      }


      .table-head { position: sticky; top: 0; background: white; }
      .total { background: #ddd; }
      .tooltip{
        position: relative;
        display: inline;
      }
      .tooltip .tooltiptext {
        width: 120px;
        visibility: hidden;
        background-color: cornsilk;
        color: black;
        text-align: center;
        padding: 5px;
        border-radius: 6px;
        /* Position the tooltip text - see examples below! */
        position: absolute;
        top: 100%;
        left: 50%;
        margin-left: -60px;
        margin-top: 10px;
        transform: translateX(-.5em);
        font-weight: normal;
        z-index: 1;
      }
      .tooltip:hover .tooltiptext {
        visibility: visible;
      }
      .tooltip .tooltiptext::after {
        content: " ";
        position: absolute;
        bottom: 100%;  /* At the top of the tooltip */
        left: 50%;
        margin-left: -5px;
        border-width: 5px;
        border-style: solid;
        border-color: transparent transparent cornsilk transparent;
      }

      .vertical-flex-container { 
        display: flex; 
        flex-direction: column; 
        align-items: stretch; 
        justify-content: space-between;
      }
      .visited { color: #800080; }

      @keyframes flash {
        0% {
          background-color: #FFF2;
          box-shadow: 32px 0 #FFF2, -32px 0 #FFF;
        }
        50% {
          background-color: #FFF;
          box-shadow: 32px 0 #FFF2, -32px 0 #FFF2;
        }
        100% {
          background-color: #FFF2;
          box-shadow: 32px 0 #FFF, -32px 0 #FFF2;
        }
      }
    '''
  }
}
