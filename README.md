const express = require("express");
const mongoose = require("mongoose");
const path = require("path");
const methodOverride = require("method-override");
const ejsMate = require("ejs-mate");
const wrapasync = require("./utiles/wrapasync");
const ExpressError = require("./utiles/expresserror");
const Listing = require("./models/listings"); // Correct path


const Review = require("./models/review");
const { listingSchema, reviewSchema } = require("./schema");
const flash = require("connect-flash");
const session = require("express-session");
const passport = require("passport");
const LocalStrategy = require("passport-local");
const User = require("./models/user");
const userrouter = require("./models/userr");
const { isloggedin } = require("./middleware");

// MongoDB connection setup
const MONGO_URL = "mongodb://127.0.0.1:27017/wondelast";
mongoose.connect(MONGO_URL)
  .then(() => console.log("Connected to MongoDB"))
  .catch(err => console.error("Failed to connect to MongoDB:", err));

// Express configurations
const app = express();
app.use(express.static(path.join(__dirname, "/public")));
app.set("view engine", "ejs");
app.set("views", path.join(__dirname, "views"));
app.use(express.urlencoded({ extended: true }));
app.use(methodOverride("_method"));
app.engine("ejs", ejsMate);

const sessionOptions = {
  secret: "mySupersecretcode",
  resave: false,
  saveUninitialized: true,
  cookie: {
    expires: Date.now() + 7 * 24 * 60 * 60 * 1000,
    maxAge: 7 * 24 * 60 * 60 * 1000,
    httpOnly: true
  }
};

app.use(session(sessionOptions));
app.use(flash());

app.use(passport.initialize());
app.use(passport.session());

passport.use(new LocalStrategy(User.authenticate()));
passport.serializeUser(User.serializeUser());
passport.deserializeUser(User.deserializeUser());

app.use((req, res, next) => {
  res.locals.success = req.flash("success");
  res.locals.error = req.flash("error");
  res.locals.curruser = req.user;
  next();
});

app.use("/", userrouter);

// Define routes
app.get("/", (req, res) => {
  res.send("hello am root");
});

// Validation middleware
const validateListing = (req, res, next) => {
  const { error } = listingSchema.validate(req.body);
  if (error) {
    const errmsg = error.details.map(el => el.message).join(",");
    console.error('Listing validation error:', errmsg);
    throw new ExpressError(400, errmsg);
  } else {
    next();
  }
};

const validateReview = (req, res, next) => {
  const { error } = reviewSchema.validate(req.body);
  if (error) {
    const errmsg = error.details.map(el => el.message).join(",");
    console.error('Review validation error:', errmsg);
    throw new ExpressError(400, errmsg);
  } else {
    next();
  }
};

// Index route
app.get("/listings", wrapasync(async (req, res) => {
  const allListings = await Listing.find({});
  res.render("listings/index", { allListings });
}));

// New route
app.get("/listings/new", isloggedin, (req, res) => {
  res.render("listings/new.ejs");
});

// Show route
app.get("/listings/:id", wrapasync(async (req, res) => {
  const { id } = req.params;
  const listing = await Listing.findById(id).populate("reviews").populate("owner");
  res.render("listings/show.ejs", { listing });
  console.log(listing);
}));

// Create route
app.post("/listings", isloggedin, validateListing, wrapasync(async (req, res, next) => {
  const newListing = new Listing(req.body.listing);
  
  newListing.owner=req.user._id;
  await newListing.save();
  req.flash("success", "New Listing Created");
  res.redirect("/listings");
}));

// Edit route
app.get("/listings/:id/edit", isloggedin, wrapasync(async (req, res) => {
  const { id } = req.params;
  const listing = await Listing.findById(id);
  res.render("listings/edit.ejs", { listing });
}));

// Update route
app.put("/listings/:id", isloggedin, validateListing, wrapasync(async (req, res, next) => {
  const { id } = req.params;
  await Listing.findByIdAndUpdate(id, { ...req.body.listing });
  res.redirect(`/listings/${id}`);
}));

// Delete route
app.delete("/listings/:id", isloggedin, wrapasync(async (req, res) => {
  const { id } = req.params;
  await Listing.findByIdAndDelete(id);
  res.redirect("/listings");
}));

// Reviews - Post route
app.post("/listings/:id/reviews", validateReview, wrapasync(async (req, res) => {
  const listing = await Listing.findById(req.params.id);
  const newReview = new Review(req.body.review);
  listing.reviews.push(newReview);
  await newReview.save();
  await listing.save();
  res.redirect(`/listings/${listing._id}`);
}));

// Delete reviews
app.delete("/listings/:id/reviews/:reviewId", wrapasync(async (req, res) => {
  const { id, reviewId } = req.params;
  await Listing.findByIdAndUpdate(id, { $pull: { reviews: reviewId } });
  await Review.findByIdAndDelete(reviewId);
  res.redirect(`/listings/${id}`);
}));

// Error handling middleware
app.all("*", (req, res, next) => {
  next(new ExpressError(404, "Page Not Found"));
});

app.use((err, req, res, next) => {
  const { statusCode = 400, message = "Something went wrong" } = err;
  res.status(statusCode).render("err.ejs", { err });
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}`);
});
