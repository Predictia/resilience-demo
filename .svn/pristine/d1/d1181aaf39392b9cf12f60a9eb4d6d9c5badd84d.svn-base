var domain = "http://178.62.156.204/";
var application = 6;
var radius = function(z){
	if(z==4) return 4;
	if(z==5) return 8;
	if(z==6) return 16;
	if(z==7) return 32;
	return 64;
}

angular.module('resilience', ['resilience.controllers','base64','ngStorage','ngDialog','resilience.services']);

angular.module('resilience.controllers', [])
.controller('HomeCtrl', function(StationService,ngDialog) {
	var initMap = function(){
		var map = L.map('map',{
			preferPixi:true,
			center:[7.38, -69.61],
			zoom:4,
			zoomControl:false,
			minZoom:4,
			maxZoom:8,
			continuousWorld:true
		});
		L.tileLayer('https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token=pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpandmbXliNDBjZWd2M2x6bDk3c2ZtOTkifQ._QA7i5Mpkd_m30IGElHziw', {
			maxZoom: 18,
			attribution: 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, ' +
				'<a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
				'Imagery © <a href="http://mapbox.com">Mapbox</a>',
			id: 'mapbox.light'
		}).addTo(map);
		var info = L.control();
		info.onAdd = function (map) {
			var div = L.DomUtil.create('div','info');
			var content = "<a href='http://www.euporias.eu' target='_blank'><img alt='EUPORIAS' title='EUPORIAS' src='euporias.png'/></a><br/>";
			content += "<h1><a href='http://resilience.euporias.eu' target='_blank'>RESILIENCE</a> outcomes</h1>";
			div.innerHTML=content;
			return div;
		};
		info.options.position='topleft';
		info.addTo(map);
		new L.Control.Zoom({position:'topleft'}).addTo(map);
		return map;
	};
	var map = initMap();
	var renderer = PIXI.autoDetectRenderer(document.body.clientWidth,document.body.clientHeight,{'transparent':true,'antialias':true});
	document.getElementById("pixi").appendChild(renderer.view);
	var stage = new PIXI.Container();
	var graphics = new PIXI.Graphics();
	graphics.interactive = false;
	stage.addChild(graphics);
	var points = [], xy = [];
	var tree;
	angular.element(document.querySelector("#loading")).css('display','block');
	StationService.GetStations().then(function(data){
		Papa.parse(data,{
			delimiter:'\t',
			header:true,
			worker:false,
			fastMode:true,
			step:function(row){
				var e = row.data[0];
				if(e.lat){
					points.push({
						point:new L.LatLng(parseFloat(e.lat),parseFloat(e.lon)),
						id:parseInt(e.cellID),
						rpss:parseFloat(e.rpss),
						meanPrediction:parseFloat(e.meanPrediction),
						meanHistoric:parseFloat(e.meanHistoric)
					});
				}
			},
			complete:function(){
				draw();
				map.on("render moveend zoomend",draw);
				map.on('resize',function(){
					renderer.resize(document.body.clientWidth,document.body.clientHeight);
					draw();
				});
				map.on("mousemove",function(x){
					var e = x.originalEvent;
					e = {x:(e.x || e.clientX),y:(e.y || e.clientY)};
					var p = resolvePoint(e);
					if(p.length){
						angular.element(document.querySelector("#map")).css("cursor","pointer");
					}else{
						angular.element(document.querySelector("#map")).css("cursor","move");
					}
					renderer.render(stage);
				});
				map.on("click",function(x){
					angular.element(document.querySelector("#loading")).css('display','block');
					var e = x.originalEvent;
					e = {x:(e.x || e.clientX),y:(e.y || e.clientY)};
					var p = resolvePoint(e);
					if(p.length){
						StationService.GetStationInfo(p[0][0].id).then(function(data){
							angular.element(document.querySelector("#loading")).css('display','none');
							ngDialog.open({
								template:'<strong>Point forecast and observations</strong>:<br/><pre>'+JSON.stringify(JSON.parse(data),null,2)+'</pre>',
								plain:true,
								showClose:true
							});
							angular.element(document.querySelector("#loading")).css('display','none');
						});
					}else{
						angular.element(document.querySelector("#loading")).css('display','none');
					}					
				});
			}
		});
	});
	var distance = function(a,b){
		return Math.sqrt(Math.pow(a.x-b.x,2)+Math.pow(a.y-b.y,2));
	}
	var draw = function(){
		graphics.clear();
		xy = [];
		for(var i=points.length-1;i>=0;i--){
			var p = points[i];
			var c = map.latLngToContainerPoint(p.point);
			xy.push({x:c.x,y:c.y,id:p.id});
			graphics.beginFill(0xe74c3c, Math.min(1,p.rpss+0.3));			
			graphics.drawCircle(c.x,c.y,radius(map.getZoom()));
			graphics.endFill();
		}
		tree = new kdTree(xy,distance,['x','y']);
		renderer.render(stage);
		angular.element(document.querySelector("#loading")).css('display','none');
	}
	var resolvePoint = function(e){
		return tree.nearest(e,1,radius(map.getZoom()));
	}
});

angular.module('resilience.services', [])
.factory('OauthService',function($http,$q,$base64,$localStorage){
	var user = "RESILIENCE.ro";
	var secret = "4e975e24-2124-4572-8b74-f6264907093e";
	$http.defaults.useXDomain = true;
	delete $http.defaults.headers.common['X-Requested-With'];
	return {
		token:function(){
			var d = $q.defer();
			var key = "resilience_access_token";
			var access_token = $localStorage[key];
			if((typeof access_token=='undefined' || angular.equals({},access_token)) || Date.now() > (access_token.expires_at - (600*1000))){
				var postdata = 'grant_type=client_credentials&scope=read';
				var authdata = "Basic "+Base64.encode(user + ':' + secret);
				$http.post(domain+'oauth/token',postdata,{headers:{'Authorization':authdata,'Content-Type':'application/x-www-form-urlencoded'}})
					.then(function(response){
						$localStorage[key] = {
							key:response.data.access_token,
							expires_at:Date.now()+(response.data.expires_in*1000)
						};
						d.resolve(response.data.access_token);
					});
			}else{
				d.resolve(access_token.key);
			}
			return d.promise;
		}
	};
})
.factory('StationService',function($http,$q,OauthService,$localStorage){
	var queryMaxForecast = function(token){
		var d = $q.defer();
		$http.defaults.headers.common.Authorization = "Bearer "+token;
		getdata = "application="+application+"&product=Global Forecast&parameter=forecastStartTime";
		$http.get(domain+'outcomes/search/metadata?'+getdata)
			.then(function(response){
				var forecast = response.data._embedded.strings[0].content;
				d.resolve({token:token,forecast:forecast});
		});
		return d.promise;
	};
	var queryStations = function(token,forecastStartTime){
		var d = $q.defer();
		var key = "resilience_forecast-"+forecastStartTime;
		var globalForecast = $localStorage[key];
		if(typeof globalForecast=='undefined' || angular.equals({},globalForecast)){
			$http.defaults.headers.common.Authorization = "Bearer "+token;
			getdata = "application="+application+"&product=Global Forecast&forecastStartTime="+forecastStartTime;
			$http.get(domain+'outcomes/search/parameters?'+getdata)
				.then(function(response){
					$localStorage[key] = response.data._embedded.outcomes;
					d.resolve(response.data._embedded.outcomes);
			});
		}else{
			d.resolve(globalForecast);
		}
		return d.promise;
	};
	var queryStationInfo = function(token,station){
		var d = $q.defer();
		$http.defaults.headers.common.Authorization = "Bearer "+token;
		getdata = "application="+application+"&product=Point Details&stationId="+station;
		$http.get(domain+'outcomes/search/parameters?'+getdata)
			.then(function(response){
				d.resolve(response.data._embedded.outcomes);
		});
		return d.promise;
	};
	return {
		GetStations:function(){
			return OauthService.token()
				.then(function(data){return queryMaxForecast(data)})
				.then(function(data){return queryStations(data.token,data.forecast)})
				.then(function(data){return Base64.decode(data[0].results[0])});
		},
		GetStationInfo:function(station){
			return OauthService.token()
			.then(function(token){
				return queryStationInfo(token,station);
			}).then(function(result){
				return Base64.decode(result[0].results[0]);
			});
		}
	}
});
