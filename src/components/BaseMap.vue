<template>
  <div class="map-wrapper">
    <GeoFilterView class="filter-overlay" @saveFilter="handleFilterData"></GeoFilterView>
    <div id="map" class="map-container"></div>
    <div id="popup" class="ol-popup">
      <div id="popup-content"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { Map, View, Feature, Overlay } from 'ol';
import { Tile as TileLayer } from 'ol/layer';
import { OSM } from 'ol/source';
import { Vector as VectorLayer } from 'ol/layer';
import { Vector as VectorSource } from 'ol/source';
import { Point, LineString } from 'ol/geom';
import { Icon, Style, Stroke } from 'ol/style';
import axios from 'axios';
import IconStartPin from '../assets/IconStartPin.png';
import IconEndPin from '../assets/IconEndPin.png';
import GeoFilterView from "@/views/GeoFilterView.vue";
import {useToast} from "vue-toastification";
import {boundingExtent} from "ol/extent";
import {fetchPersonById, fetchPersons} from "@/services/apiService.js";

const toast = useToast();

let projection = ref("EPSG:4326");
let center = ref([-60.457873, 0.584053]);
let zoom = ref(5);
let map = ref<Map | null>(null);
let pointFeatures = ref<Feature[]>([]);
let routeLine = ref<Feature[]>([]);
let pointFinalStar = ref<Feature[]>([]);
let lineLayer = ref<VectorLayer<VectorSource> | null>(null);
let thePerson:{person: number | null, startDate:string | null, endDate:string | null};
let PersonOption = ref<{ label: string; value: number }[]>([]);


function handleFilterData(filterData:{person: number | null, startDate:string | null, endDate:string | null}){
  pointFeatures.value = [];
  map.value.removeLayers;
  routeLine.value = [];
  pointFinalStar.value = [];

  let getUrl = `http://localhost:8080/tracker/period/${filterData.person}/${filterData.startDate}T00:00:00.000/${filterData.endDate}T00:00:00.000?page=0`;

  getAllPoints(getUrl).then((points) => {
    if (points.length === 0) {
      toast.info("Nenhum ponto encontrado para o filtro selecionado.");
    } else {
      let pointList = new ref(points);
      makeGeometryPointFromArray(pointList, filterData.person);
      lineLayer.value = makeLineFromPoints(pointFeatures);
      map.value.addLayer(lineLayer.value);
      adjustMap();
    }
  });
}

const getAllPoints = async (getPointsUrl: string) => {
  try {
    const response = await axios.get(getPointsUrl);
    return response.data.content;
  } catch (error) {
    console.error(error);
    toast.error("Erro ao buscar pontos. Tente novamente mais tarde.");
  }
};

function createStartLayer(pointFinalStarArrayOfFeatures) {
  const vectorLayer = new VectorLayer({
    source: new VectorSource({
      features: pointFinalStarArrayOfFeatures.value,
    }),
    zIndex: 2,
  });
  map.value.addLayer(vectorLayer);
}

function makeGeometryPointFromArray(arrayOfGeometryObjects, nameFilter?) {
  if (arrayOfGeometryObjects.length === 0) return [];

  if (nameFilter) {
    const startPoint = new Feature({
      geometry: new Point([arrayOfGeometryObjects.value[0].longitude, arrayOfGeometryObjects.value[0].latitude]),
      name: 'Início do percurso' // Adiciona o title nativo
    });

    startPoint.setStyle(new Style({
      image: new Icon({
        src: IconStartPin,
        scale: 0.7,
        anchor: [0.5, 1],
      }),
    }));
    startPoint.setProperties({
      personName: nameFilter
    })

    const endPoint = new Feature({
      geometry: new Point([arrayOfGeometryObjects.value[arrayOfGeometryObjects.value.length - 1].longitude, arrayOfGeometryObjects.value[arrayOfGeometryObjects.value.length - 1].latitude]),
      name: 'Fim do percurso' // Adiciona o title nativo

    });

    endPoint.setStyle(new Style({
      image: new Icon({
        src: IconEndPin,
        scale: 0.7,
        anchor: [0.5, 1],
      }),
    }));
    endPoint.setProperties({
      personName: nameFilter
    })

    pointFinalStar.value.push(startPoint);
    pointFinalStar.value.push(endPoint);
    createStartLayer(pointFinalStar);

    center.value = endPoint.getGeometry().getCoordinates();
  }

  arrayOfGeometryObjects.value.forEach((pointObj) => {
    const point = new Feature({
      geometry: new Point([pointObj.longitude, pointObj.latitude]),
    });
    point.setStyle(new Style({
      image: new Icon({
        src: IconEndPin,
        scale: 0.7,
        anchor: [0.5, 1],
      }),
    }));
    pointFeatures.value.push(point);
  });
}

function makeLineFromPoints(featureList) {
  if (featureList.value.length === 0) {
    console.log('Nenhum ponto disponível para criar linhas');
    toast.info("Nenhum ponto disponível para criar linhas.");
    return null;
  }

  const groupedById = featureList.value.reduce((acc, feature) => {
    const idText = feature.get('idText');
    if (!acc[idText]) acc[idText] = [];
    acc[idText].push(feature);
    return acc;
  }, {});

  Object.keys(groupedById).forEach((idText) => {
    const points = groupedById[idText];
    if (points.length >= 2) {
      for (let i = 0; i < points.length - 1; i++) {
        const point1 = points[i];
        const point2 = points[i + 1];

        const lineFeature = new Feature({
          geometry: new LineString([point1.getGeometry().getCoordinates(), point2.getGeometry().getCoordinates()]),
        });
        lineFeature.setStyle(new Style({
          stroke: new Stroke({
            color: '#ec1c24',
            width: 6,
          }),
        }));
        routeLine.value.push(lineFeature);
      }
    }
  });
  return new VectorLayer({
    source: new VectorSource({
      features: routeLine.value,
    })
  })
}

const adjustMap = () => {
  const coordinates = pointFeatures.value.map((pontos) =>
      pontos.getGeometry().getCoordinates()
  );
  const extent = boundingExtent(coordinates);
  if (map.value) {
    map.value
        .getView()
        .fit(extent, {padding: [50, 50, 50, 50], maxZoom: 15});
  }
};

const createMap = () => {
  map.value = new Map({
    target: 'map',
    layers: [
      new TileLayer({
        source: new OSM(),
      }),
    ],
    view: new View({
      center: center.value,
      zoom: zoom.value,
      projection: projection.value,
    }),
  });

  const vectorLayer = new VectorLayer({
    source: new VectorSource({
      features: pointFeatures.value,
    }),
  });

  const routeLayer = new VectorLayer({
    source: new VectorSource({
      features: routeLine.value,
    }),
  });

  map.value.addLayer(vectorLayer);
  map.value.addLayer(routeLayer);
}

async function getPersonName(personID: number): Promise<{ label: string; value: number } | undefined> {
  let personListFromDb = await fetchPersons();

  
  // Atribui a lista de opções com a tipagem correta
  PersonOption.value = personListFromDb.map(person => ({
    label: person.fullName.toUpperCase(),
    value: person.idPerson
  }));

  // Retorna a pessoa correspondente ao ID
  return PersonOption.value.find(person => person.value.idPerson === personID);
}

onMounted(() => {

  createMap()
  
  const popupElement = document.getElementById('popup') as HTMLElement;
  const popupContent = document.getElementById('popup-content') as HTMLElement;
  const popupOverlay = new Overlay({
    element: popupElement,
    autoPan: true,
    autoPanAnimation: {
      duration: 250,
    },
  });
  map.value?.addOverlay(popupOverlay);
  
  // Adiciona o evento de clique ao mapa
  map.value?.on('click', (e) => {
  let clickedFeature = false;

  map.value?.forEachFeatureAtPixel(e.pixel, (feature) => {
    const featureName = feature.get('name');

    if (featureName) {
      clickedFeature = true;
      const coordinates = feature.getGeometry().getCoordinates();
      const offsetCoordinates = [coordinates[0], coordinates[1]];
      popupOverlay.setPosition(offsetCoordinates);

      // Obtenha o nome da pessoa usando a Promise corretamente
      getPersonName(thePerson)
        .then(personName => {
          // Verifique se a pessoa foi encontrada e atualize o conteúdo do popup
          popupContent.innerHTML = `<p>${personName?.label}</p>`;
          popupElement.style.display = 'block';
        })
        .catch(error => {
          console.error(error);
          popupContent.innerHTML = '<p>Erro ao buscar o nome</p>';
          popupElement.style.display = 'block';
        });
    }
  });

  if (!clickedFeature) {
    popupOverlay.setPosition(undefined);
    popupElement.style.display = 'none';
  }
});
});



</script>

<style scoped>
.map-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  z-index: 1;
}
.filter-overlay {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  background-color: white;
  padding: 10px;
  border-radius: 8px;
}
.ol-popup {
  position: absolute;
  background-color: rgb(80, 80, 80);
  padding: 10px;
  border: 1px solid black;
  border-radius: 8px;
  min-width: 150px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
  display: none;
  pointer-events: none; /* Desativa interações quando o popup está escondido */
}

.ol-popup.active {
  display: block;
  pointer-events: auto; /* Ativa interações quando o popup está visível */
}



</style>