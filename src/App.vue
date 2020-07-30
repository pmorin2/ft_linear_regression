<template>
  <div id="app">
    <table class="gridInfo" v-if="trainingScreen === true">
      <thead>
      <tr>
        <th>THETA0</th>
        <th>THETA1</th>
      </tr>
      </thead>
      <tbody>
      <tr>
        <td>{{ this.theta0 }}</td>
        <td>{{ this.theta1 }}</td>
      </tr>
      </tbody>
    </table>
    <table class="gridData" v-if="trainingScreen === true">
      <thead>
      <tr>
        <th>KILOMÉTRAGE</th>
        <th>PRIX</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="entry in this.dataCSV" :key="entry.km">
        <td>{{realValue(entry.km)}}</td>
        <td>{{realValue(entry.price)}}</td>
      </tr>
      </tbody>
    </table>
    <img alt="Vue logo" src="./assets/logo.png"><br /><br />
    <el-upload
            v-show="trainingScreen === false"
            id="upload"
            class="upload-demo"
            action="https://jsonplaceholder.typicode.com/posts/"
            :on-success="loadCSV"
            :on-remove="CSVremove"
            :limit="1"
            accept="text/csv">
      <el-button size="small" type="info">Cliquer pour envoyer un fichier <i class="el-icon-upload el-icon-right"></i></el-button>
      <div slot="tip" class="el-upload__tip">Fichiers csv uniquement</div>
    </el-upload><br />
    <el-button class="actionButton" type="success" @click="estimate">Faire une estimation</el-button>
    <el-button class="actionButton" v-show="trainingScreen === false" type="success" size="medium" @click="trainingScreenOn">Lancer le training</el-button>
    <line-chart v-if="trainingScreen === true" :chart-data="dataCollection"></line-chart>
  </div>
</template>

<script>

import LineChart from "./LineChart";

export default {
  name: 'App',
  components: {LineChart},
  data() {
    return {
      count: 0,
      theta0: 0,
      norm: 1,
      cost: [],
      theta1: 0,
      dataCSV: null,
      parse_header: [],
      learningRate: 0.001,
      trainingScreen: false,
      dataCollection: null,
    }
  },
  methods: {
    csvJSON(csv) {
      let result = [];
      let lines = csv.split("\n");
      let headers = lines[0].split(",");
      this.parse_header = lines[0].split(",");

      lines.map(function(line, indexLine) {
        if (indexLine < 1) return ;

        let obj = {};
        let currentline = line.split(",");

        headers.map(function(header, indexHeader) {
          obj[header] = currentline[indexHeader]
        });

        result.push(obj)
      });

      result.pop();
      return result;
    },

    loadCSV(res, file) {
      let elem = this;
      if (window.FileReader) {
        var reader = new FileReader();
        reader.readAsText(file.raw);

        reader.onload = function(event) {
          let csv = event.target.result;
          elem.dataCSV = elem.csvJSON(csv).sort(function (a, b) {
            return a["km"] - b["km"] || a["price"] - b["price"];
          })
        };
      }
    },

    CSVremove() {
      this.dataCSV = null;
    },

    prixEstime(kilometrage) {
      return (this.theta0 + (this.theta1 * kilometrage))
    },

    calc_s0() {
      let sum = 0;
      let thisElem = this;
      this.dataCSV.forEach(function (element) {
        sum += thisElem.prixEstime(element.km) - element.price;
      });
      return(sum * this.learningRate / this.dataCSV.length)
    },

    calc_s1() {
      let sum = 0;
      let tmp = 0;
      let thisElem = this;
      this.dataCSV.forEach(function (element) {
        tmp = thisElem.prixEstime(element.km) - element.price;
        sum += tmp * element.km;
      });
      return(sum * this.learningRate / this.dataCSV.length)
    },

    /*eslint no-constant-condition: ["error", { "checkLoops": false }]*/
    training() {
      let tmpTheta0 = 0;
      let tmpTheta1 = 0;
      while (true) {
        tmpTheta0 = this.calc_s0();
        tmpTheta1 = this.calc_s1();
        this.theta0 -= tmpTheta0;
        this.theta1 -= tmpTheta1;
        this.calculate_cost();
        if (this.cost[this.cost.length - 2] - this.cost[this.cost.length - 1] < 0.000000005) {
          break
        }
      }
      this.theta0 *= this.norm;
      this.trainingScreen = true;
      this.$message({
        showClose: true,
        message: "Nombre d'itérations réalisées :".concat(" ", this.cost.length),
      });
      this.dataCollection = {
        labels: this.onlyKm(),
        datasets: [
          {
            label: 'Data set',
            backgroundColor: 'transparent',
            borderColor: '#f87979',
            pointBackgroundColor: "#842bd7",
            data: this.onlyPrice()
          },
          {
            label: 'Estimated Data',
            backgroundColor: 'transparent',
            borderColor: '#05a10b',
            pointBackgroundColor: "#842bd7",
            data: this.onlyEstimated()
          }
        ]
      }
    },

    onlyKm() {
      let kms = [];
      let thisElem = this;
      this.dataCSV.forEach(function (element) {
        kms.push(Math.round(element.km * thisElem.norm));
      });
      return kms;
    },

    onlyPrice() {
      let prices = [];
      let thisElem = this;
      this.dataCSV.forEach(function (element) {
        prices.push(Math.round(element.price * thisElem.norm));
      });
      return prices;
    },

    onlyEstimated() {
      let thisElem = this;
      let estimations = [];
      this.onlyKm().forEach(function (element) {
        estimations.push(Math.round(thisElem.prixEstime(element)));
      });
      return estimations;
    },

    realValue(value) {
      return Math.round(value * this.norm)
    },

    trainingScreenOn() {
      if (this.dataCSV !== null) {
        this.normalize_data();
        this.calculate_cost();
        this.training();
      }
      else {
        this.$message({
          showClose: true,
          message: "aucun csv n'a été upload",
          type: 'error',
        });
      }
    },

    normalize_data() {
      let thisElem = this;
      let kms = [];
      let kmMax = 0;
      this.dataCSV.forEach(function (element) {
        kms.push(element.km);
      });
      kmMax = Math.max(...kms);
      while (kmMax > 100) {
        this.norm *= 10;
        kmMax /= 10;
      }
      this.dataCSV.forEach(function (element, index) {
        thisElem.dataCSV[index].price /= thisElem.norm;
        thisElem.dataCSV[index].km /= thisElem.norm;
      });
    },

    calculate_cost() {
      let thisElem = this;
      let step_1 = 0;
      let sum = 0;
      this.dataCSV.forEach(function (element) {
        step_1 = thisElem.prixEstime(element.km) - element.price;
        sum += step_1 * step_1;
      });
      this.cost.push(sum / 2 * this.dataCSV.length);
    },

    async estimate() {
      let kilometrage;
      kilometrage = parseInt(await prompt("kilométrage de la voiture", "0"));
      if (!kilometrage) {
        this.$message({
          showClose: true,
          message: "kilométrage invalide",
          type: 'error',
        });
      }
      else if (this.prixEstime(kilometrage) < 0) {
        this.$message({
          showClose: true,
          message: "Résultat éroné: l'algorithme n'est pas assez entrainé pour des valeurs si élevées",
          type: 'error',
        });
      }
      else {
        this.$message({
          showClose: true,
          message: "prix estimé :".concat(" ", Math.round(this.prixEstime(kilometrage)).toString()),
        });
      }
    },
  }
}
</script>

<style>
#app {
  font-family: Lato, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}
.actionButton {
  margin-top: 20px !important;
  font-size: 25px !important;
  margin-bottom: 60px !important;
}
.gridData {
  position: absolute;
  right: 5%;
}

.gridInfo {
  position: absolute;
  left: 5%;
}

table {
  border: 2px solid #42b983;
  border-radius: 3px;
  background-color: #fff;
}

th {
  background-color: #42b983;
  color: #ffffff;
  font-weight: 900;
}

td {
  background-color: #f9f9f9;
  font-weight: 700;
}

th,
td {
  min-width: 120px;
  padding: 10px 10px;
}
</style>
