<script>
import {
  onMount
} from "svelte";
import LinkButton from "../LinkButton.svelte";
import {
  api
} from "../rest/config";
import {
  fundNames,
  fundRights,
  goTo,  
  isWriter,
  user
} from "../rest/store";
import {
  getTable
} from '../rest/table.request'

let error = ""
let query
let order = "desc" 
let where = ""
let timeout
$: fundOptions = []
let currentFund = ""
async function tableRequest() {
  const res = await getTable(query, order);
  if (res.status !== 'OK') return error = res.msg || res
  return res
}

function mountTable(data = [], last_page) {
  // Preparations
  const arrayFormatter = (cell) => {
    const {
      field
    } = cell._cell.column;

    const content = cell._cell.row.data[field].join(", ");
    const div = document.createElement("div");
    div.title = content;
    div.innerText = content;
    return div;
  };
  const arrayField = (name, label) => {
    return {
      title: label,
      field: name,
      formatter: arrayFormatter,
      headerSort: false,
    };
  };
  // Table mounting
  new Tabulator("#tableMountingPoint", {
    data,
    pagination: "remote", //enable remote pagination
    paginationSize: 25,
    paginationSizeSelector: [10, 25, 50, 100],
    ajaxURL: api + "table", //set url for ajax request
    ajaxSorting: true, //send sort data to the server instead of processing locally
    ajaxParams: {
      where: where,
      // id: query || "",
      // order: order,
      author: currentFund
    },
    columnMaxWidth: 300,
    dataTree: true,
    headerSort: false,
    // groupBy: "fundName",

    columns: [{
        title: "<div class='editColumnTitle'></div>",
        field: "id",
        frozen: true,
        minWidth: 0,
        width: 66,
        formatter: function(cell) {
          const id = cell._cell.row.data.id;
          const author = cell._cell.row.data.author;
          const a = document.createElement("a")
          a.className = "linkToRow material-icons notranslate";  
          a.innerText = "create";
          a.style.textDecoration = 'none'

          // Case root && has parent
          if (cell._cell.row.data.parent && $user.role === "root") {      
            a.href = "./editNote?id=" + id + "&parent=" + cell._cell.row.data.parent
            return a;
          }
          
          // Case can edit
          if ( $user.role === "root" ||
          // fund member, not readonly, not history
            ( $fundRights[author] && $fundRights[author] !== "readonly" && !cell._cell.row.data.parent)
          ) {
            a.href = "./editNote?id=" + id
            return a
          }

          // Case draw modal
          const i = document.createElement("i")
          i.className = "linkToRow material-icons notranslate"
          i.innerText = "visibility"         
          i.addEventListener("click", () => {
            mountModal(id, cell._cell.row.data)
          });
          return i
        },
        headerSort: false,
      },
      {
        title: "Фонд",
        field: "author",
        headerSort: false,
        formatter: (cell) => {
          const author = cell._cell.row.data.author
          return $fundNames[author]
        },
      },
      {
        title: "Создано",
        field: "created",
        formatter: (cell) => {
          if (cell._cell.row.data.old) {
            return "архив";
          } else
            return moment(cell._cell.row.data.created).format("MM-DD-YYYY");
        },
        headerSort: true,
      },
      {
        title: "Обновлено",
        field: "updated",
        formatter: (cell) => {
          if (cell._cell.row.data.old) {
            return "архив";
          } else
            return moment(cell._cell.row.data.updated).format("MM-DD-YYYY");
        },
        headerSort: true,
      },
      {
        title: "Автор",
        field: "fundName",
        visible: false,
        sorter: "string",
      },
      {
        title: "Арбитраж",
        field: "case",
        width: 150,
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.case.map((caseObj) => {
            return caseObj.arbitrage;
          });
          const content = mappos.join(", ");
          const div = document.createElement("div");
          div.title = content;
          div.innerText = content;
          return div;
        },
        sorter: "string",
      },
      {
        title: "Ники",
        field: "nickname",
        width: 350,
        headerSort: false,
        formatter: function(cell) {
          const mappos = cell._cell.row.data.nickname.map((obj) => {
            if (cell._cell.row.data.old)
              return cell._cell.row.data.nicknameOld;
            else if (obj.room)
              return obj.room + (obj.value && " - " + obj.value);
            else return obj.value;
          });
          const content = mappos.join(", ");
          const div = document.createElement("div");
          div.title = content;
          div.innerText = content;
          return div;
        },
      },
      {
        title: "Дисциплина",
        field: "nickname",
        width: 150,
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.nickname.map((nickObj) => {
            return nickObj.discipline; // !
          });
          return mappos.join(", ");
        },
        sorter: "string",
      },
      {
        title: "ФИО",
        field: "FIO",
        width: 250,
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.FIO.map((fioObj) => {
            return (
              (fioObj.lastname || "") +
              " " +
              (fioObj.firstname || "") +
              " " +
              (fioObj.middlename || "")
            );
          });
          return mappos.join(", ");
        },
      },
      {
        title: "Описание",
        field: "case",
        width: 250,
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.case.map((caseObj) => {
            return caseObj.descr;
          });
          const content = mappos.join(" | ");
          const div = document.createElement("div");
          div.title = content;
          div.innerText = content;
          return div;
        },
        sorter: "string",
      },
      {
        title: "Ущерб ($)",
        field: "case",
        width: 100,
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.case.map((caseObj) => {
            return caseObj.amount;
          });
          const content = mappos.join(" | ");
          const div = document.createElement("div");
          div.title = content;
          div.innerText = content;
          return div;
        },
        sorter: "string",
      },
      arrayField("gipsyteam", "Gipsy team"),
      arrayField("skype", "Skype"),
      arrayField("skrill", "Skrill"),
      arrayField("neteller", "Neteller"),
      arrayField("phone", "Телефоны"),
      {
        title: "Адреса",
        field: "location",
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.location.map((obj) => {
            return obj.country + " " + obj.town + " " + obj.address;
          });
          return mappos.join(" ,");
        },
      },
      arrayField("pokerstrategy", "Poker Strategy"),
      arrayField("google", "Google"),
      arrayField("mail", "e-mail"),
      arrayField("vk", "Вконтакте"),
      arrayField("facebook", "Facebook"),
      arrayField("blog", "Блог"),
      arrayField("forum", "Форум"),
      arrayField("instagram", "Instagram"),
      arrayField("ecopayz", "Ecopayz"),
      {
        title: "Webmoney",
        field: "webmoney",
        headerSort: false,
        formatter: function(cell, formatterParams, onRendered) {
          const mappos = cell._cell.row.data.webmoney.map((obj) => {
            const wallets =
              obj.wallets &&
              Array.isArray(obj.wallets) &&
              obj.wallets.join(" ,");
            return obj.WMID + ": " + wallets;
          });
          return mappos.join(" | ");
        },
      },
      {
        title: "Комментарии",
        field: "comments",
        headerSort: false,
      },
    ],

    locale: true,
    langs: {
      "en-gb": {
        columns: {},
        ajax: {
          loading: "Loading",
          error: "Error",
        },
        groups: {
          item: "item",
          items: "items",
        },
        pagination: {
          page_size: "Page Size",
          page_title: "Show Page",
          first: "First",
          first_title: "First Page",
          last: "Last",
          last_title: "Last Page",
          prev: "Prev",
          prev_title: "Prev Page",
          next: "Next",
          next_title: "Next Page",
          all: "All",
        },
        headerFilters: {
          default: "filter column...",
          columns: {
            column: "filter name..."
          },
        },
        custom: {},
      },
      "ru-ru": {
        columns: {},
        ajax: {
          loading: "Загрузка...",
          error: "Ошибка!",
        },
        groups: {
          item: "свойство",
          items: "свойства",
        },
        pagination: {
          page_size: "Кол-во строк",
          page_title: "Показать страницу",
          first: "Первая",
          first_title: "Первая страница",
          last: "Последняя",
          last_title: "Последняя страница",
          prev: "Пред.",
          prev_title: "Предыдущая страница",
          next: "След.",
          next_title: "Следущая страница",
          all: "Всё",
        },
        headerFilters: {
          default: "Отфильтровать...",
          columns: {},
        },
        custom: {},
      },
    },
    tableBuilt: function() {
      document.querySelector(".tabulator").classList.add("compact", "very");
      this.redraw(true);
    },
  });

}

function mountModal(id, data) {

  const modalConfig = {
    modal: true,
    fields: {
      author: {
        label: "Автор",
        type: "select",
        options: fundOptions,
        readonly: true,
        value: $fundNames[id]
      },

      // Rest info

      case: {
        label: "Арбитраж",
        type: "multiple",
        value: [],
        settings: {
          arbitrage: {
            label: "Арбитраж",
            row: 1,
          },
          descr: {
            label: "Описание",
            type: "textarea",
            row: 1,
          },
          amount: {
            label: "Размер",
            row: 1,
          },
        },
        visible: data.case.length
      },
      
      nicknameOld: {
        label: "Архивные дицсциплины и никнеймы",
        type: "textarea",
        visible: data.nicknameOld,
      },

      nickname: {
        label: "Дисциплины",
        type: "multiple",
        value: [],
        settings: {
          discipline: {
            label: "Дисциплина",
            row: 1,
          },
          room: {
            label: "Room",
            row: 1,
          },
          value: {
            label: "Nick",
            row: 1,
          },
        },
        visible: data.nickname.length
      },



      FIO: {
        label: "ФИО",
        type: "multiple",
        value: [],
        settings: {
          firstname: {
            label: "Имя",
            row: 1,
          },
          lastname: {
            label: "Фамилия",
            row: 1,
          },
          middlename: {
            label: "Отчество",
            row: 1,
          },
        },
        visible: data.FIO.length
      },


      gipsyteam: {
        label: "Gipsy team",
        type: "creatable",
        outlined: true,
        visible: data.gipsyteam.length
      },
      skype: {
        label: "Аккаунты Skype",
        type: "creatable",
        outlined: true,
        visible: data.skype.length
      },
      skrill: {
        label: "Аккаунты skrill",
        type: "creatable",
        outlined: true,
        visible: data.skrill.length
      },
      neteller: {
        label: "Аккаунты neteller",
        type: "creatable",
        outlined: true,
        visible: data.neteller.length
      },
      phone: {
        label: "Телефоны",
        type: "creatable",
        outlined: true,
        visible: data.phone.length
      },
      pokerstrategy: {
        label: "Poker Strategy",
        type: "creatable",
        outlined: true,
        visible: data.pokerstrategy.length
      },
      google: {
        label: "Google аккаунты",
        type: "creatable",
        outlined: true,
        visible: data.google.length
      },
      mail: {
        label: "Адреса e-mail",
        type: "creatable",
        outlined: true,
        visible: data.mail.length
      },
      vk: {
        label: "Аккаунты vkontakte",
        type: "creatable",
        outlined: true,
        visible: data.vk.length
      },
      facebook: {
        label: "Аккаунты facebook",
        type: "creatable",
        outlined: true,
        visible: data.facebook.length
      },
      blog: {
        label: "Блоги",
        type: "creatable",
        outlined: true,
        visible: data.blog.length
      },
      instagram: {
        label: "Аккаунты instagram",
        type: "creatable",
        outlined: true,
        visible: data.instagram.length
      },
      forum: {
        label: "Форумы",
        type: "creatable",
        outlined: true,
        visible: data.forum.length
      },
      ecopayz: {
        label: "Аккаунты ecopayz",
        type: "creatable",
        outlined: true,
        visible: data.ecopayz.length
      },
      location: {
        label: "Адреса",
        type: "multiple",
        value: [],
        settings: {
          country: {
            label: "Страна",
            row: 1,
          },
          town: {
            label: "Город",
            row: 1,
          },
          address: {
            label: "Адрес",
            row: 1,
          },
        },
        visible: data.location.length
      },
      webmoney: {
        label: "Аккаунты Webmoney",
        type: "multiple",
        value: [],
        settings: {
          WMID: {
            row: 1,
            label: "WMID",
            required: true,
          },
          wallets: {
            label: "Кошельки",
            type: "creatable",
            row: 1,
          },
        },
        visible: data.webmoney.length
      },
      comments: {
        label: "Комментарии",
        type: 'textarea',
        visible: data.comments
      },
    },
    title: "Просмотр записи",
    buttons: null,
    noButtons: true,
    global: {
      fields: {
        readonly: true
      }
    }
  };
  window.callForm2("#modalPoint", data, modalConfig);
}

async function onFilter(e) {
  let input = e.target.value
  clearTimeout(timeout)
  timeout = setTimeout(() => {
    where = input;
    mountTable()
  }, 500);
}
onMount(async () => {
  const res = await tableRequest()
  if (res) mountTable(res.data, res.last_page)
  const temp = [{id: "", name: 'Все фонды'}]
  Object.entries($fundNames).forEach(([id,name])=> temp.push({id,name}))
  fundOptions = temp
});
function onBlur(e) {
  mountTable()
}
</script>

<main>
  {#if error}
    {error}
    <br />
  {/if}
  <div class="tableControls">
    <div class="userControls inline ui input">
      <!-- svelte-ignore a11y-no-onchange -->
      <select bind:value={currentFund} on:change={onBlur} class="item">
        {#each fundOptions as fund}
          <option value={fund.id} class="item">
            {fund.name}
          </option>
        {/each}
      </select>

      <div class="ui icon input item">
        <input type="text" on:input={onFilter} placeholder="Поиск записей" />
        <i class="search link icon" />
      </div>

      {#if $user.role === "root" || isWriter()}
        <div class="createContainer item">
          <LinkButton to="createNote" label="Создать запись"/>
        </div>
      {/if}
    </div>
  </div>

  <div id="tableMountingPoint" />
  <div id="modalPoint" />
</main>

<style>
  .userControls {
    padding: 6px;
  }

  .userControls input {
    min-width: 280px;
  }

  .inline {
    flex-wrap: wrap;
  }

  .createContainer {
    display: inline-block;
    margin-left: 24px;
  }
  select{
    padding: 0 6px;
    border: 1px solid rgb(223, 223, 223);
    border-radius: 2px;
  }
  .item{
    margin: 0 8px;
  }
</style>
